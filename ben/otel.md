# OTel Notes

## Mental Model

Spans are structured, timed log entries that form a tree matching your call graph.

```
HTTP Request arrives
├── [Span: handle-request]        ← start stopwatch #1
│   ├── [Span: validate-input]    ← start stopwatch #2, stop when done
│   ├── [Span: query-db]          ← start stopwatch #3, stop when done
│   └── [Span: format-response]   ← start stopwatch #4, stop when done
│   └── stop stopwatch #1
```

Three core concepts:

| Concept | Analogy | What it is |
|---------|---------|------------|
| **Trace** | The whole receipt | One end-to-end request, identified by a single trace ID |
| **Span** | One line item | A single operation: start time, end time, name, metadata |
| **Context** | The thread connecting them | How parent-child relationships are tracked automatically |

The application logic is just:

1. "I'm starting something" → start span
2. "Here's some info" → set attribute / add event
3. "It went wrong" → record exception
4. "I'm done" → end span (automatic with `with`)

Nesting `with` blocks auto-links parent→child via context propagation. Everything else is infrastructure to collect and ship spans.

## Span

A span is two things in one object:

1. **Context carrier** — holds `SpanContext` (trace_id + span_id + trace_flags + trace_state). This is what propagates: to child spans, across async boundaries, across processes (via W3C `traceparent` header). Always works, even if the span records nothing.
2. **Data recorder** — stores attributes, events, links, status, timing. This part is optional (can be disabled by sampling or no-SDK mode).

These are independent. A non-recording span still carries identity and parentage — child spans still know their parent, the trace_id still flows end-to-end.

## `span.is_recording()`

A span can exist but not record data. Two cases:

1. **Sampled out** — SDK's Sampler decided not to record this span. The span still propagates context (trace_id flows to children), but drops all attributes/events.
2. **No-op span** — No SDK configured (API-only mode). `NonRecordingSpan` propagates context, records nothing.

`set_attribute()` on a non-recording span is a silent no-op. The check exists purely as a **cost gate** — skip expensive prep work (JSON serialization, dict construction) when no one will consume the result.

```python
# Pattern in MAF observability.py:
if SENSITIVE_DATA_ENABLED and messages and span.is_recording():
    _capture_messages(...)  # expensive serialization — skip if span won't record
```

## TracerProvider Architecture

```
TracerProvider (singleton, global via trace.set_tracer_provider())
├── Tracer registry (name/version → Tracer)
│   ├── Tracer: "agent_framework" v1.0  ──┐
│   └── Tracer: "openai" v2.3           ──┤
│                                          │  start_span() calls back
│                                          ▼  into provider
└── SpanProcessor list ◄───────────────────┘
    ├── BatchSpanProcessor
    │   └── OTLPSpanExporter → OTLP collector
    └── BatchSpanProcessor
        └── AzureMonitorTraceExporter → App Insights
```

### Roles

| Component | Role | Customization needed? |
|-----------|------|----------------------|
| **TracerProvider** | Singleton registry. Holds tracers + processors. Global via `trace.set_tracer_provider()`. | Rarely — just instantiate and configure once at startup |
| **Tracer** | Named span factory (`name` + `version`). Creates spans tagged with instrumentation scope. No export logic. | Never — just call `get_tracer("lib_name")` |
| **SpanProcessor** | Receives finished spans from provider, batches/filters, forwards to exporter. | Rarely — `BatchSpanProcessor` covers 99% of cases |
| **SpanExporter** | Serializes spans into platform format and ships to backend. **This is the vendor extension point.** | Always platform-specific (Azure Monitor, Datadog, Jaeger, etc.) |

### Key design decisions

- **Tracer doesn't own export logic** — it's just a scoped factory that calls back into the provider.
- **Provider doesn't know about exporters directly** — you wrap an exporter in a processor, then register the processor via `provider.add_span_processor(BatchSpanProcessor(exporter))`.
- **Separation of concerns**: batching/retry (processor) vs. serialization/transport (exporter).
- **Vendor SDKs** (e.g. `azure-monitor-opentelemetry-exporter`) are just custom `SpanExporter` implementations. Everything upstream (Tracer, Processor, context propagation) stays generic OTel.

### Analogous pattern for Logs and Metrics

The same 3-tier pattern repeats:

| Signal | Provider | Named Handle | Processor/Reader | Exporter |
|--------|----------|-------------|------------------|----------|
| Traces | `TracerProvider` | `Tracer` | `SpanProcessor` | `SpanExporter` |
| Logs | `LoggerProvider` | `Logger` | `LogRecordProcessor` | `LogRecordExporter` |
| Metrics | `MeterProvider` | `Meter` | `MetricReader` | `MetricExporter` |

## Python `contextvars` and `otel_context.attach(ctx)/detach(token)`

`contextvars.ContextVar` is a Python stdlib primitive (not OTel-specific) that provides:
- **Task-scoped isolation** — each asyncio Task gets its own copy, so concurrent Tasks don't interfere (unlike thread-locals which are shared across all Tasks on the same event loop thread)
- **Implicit shared state** — makes values available across an entire call chain without passing them as function parameters (e.g. the current span doesn't need to be threaded through every function signature). Same idea as React's `Context.Provider` + `useContext()` avoiding prop drilling.

OTel uses a single `ContextVar` as storage for its current `Context` object (an immutable dict-like snapshot containing the current span).

`otel_context.attach(new_ctx)` — overwrites the ContextVar with `new_ctx` (containing the new span). Returns a Token that holds a reference to the old context value inside it — the token is the only way to get back to the previous state.
`otel_context.detach(token)` — reads the old context value from the token and writes it back into the ContextVar, restoring the previous span scope.

This creates a logical stack of span scopes — new spans created after `attach` auto-parent under the attached span, and `detach` restores the previous parent. The stack discipline (detach in reverse order of attach) is enforced by OTel convention, not by `contextvars` itself.

Call chain: `trace.use_span(span)` / `tracer.start_as_current_span(name)` / `_activate_span(span)` → `otel_context.attach(trace.set_span_in_context(span))` → `ContextVar.set(new_ctx)`

`_activate_span` is MAF's thin wrapper around the same attach/detach — exists so it can be passed as a per-pull context manager for streaming spans.