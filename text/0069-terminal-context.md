# Controlling auto-instrumentation via terminal distributed context

**Status**: `none`

## TL;DR

Add `CollectionMode` property on the `DistributedContext` to control instrumentation within this scope.
Current scenarios include turning any nested instrumentation off (traces, propagation and metrics) and could be extended in future to disable each signal separately.

## Motivation

In some scenarios auto-tracing needs to be restricted. This typically affects protocol-level instrumentation (HTTP/gRPC) that is applied process-wide with very little control from users (monkey-patching, byte code rewriting, re-jitting, profiler APIs, diagnostics channels/sources, etc).

A few scenarios where it is necessary are:

1. Specific client library uses standard protocol (HTTP, gRPC) for which auto-instrumentation is supported by OpenTelemetry and not controlled by the client library. The library is instrumented with OpenTelemetry and traces outgoing calls over this protocol preferring it over auto-instrumentation for reasons like
   - security and privacy: raw URLs may contain sensitive information
   - data richness: additional information (e.g. in headers) is essential for particular service/library observability.
   - ability to trace calls in languages where auto-instrumentation requires additional setup and is possible only for a subset of the customers (byte-code rewriting, profiling APIs, etc)
   - legacy custom correlation protocols support for the underlying service
2. Avoiding tracing internal operations:
   - Serverless environments may want to enable users calls tracing but never expose to customer internal operation they make.
   - In-process exporters such as Zipkin or OTLP exporter run on top of the HTTP/gRPC protocols but users are not generally interested in tracing calls to exporters
3. Customers do not want to trace certain endpoints to:
   - not expose any information including trace-context because of privacy or bandwidth, or because downstream service restricts accepted headers.
   - reduce tracing data volume for uninteresting/not important calls

## Proposal

Introduce DistributeContext `CollectionMode` property that indicates if context is terminal operation to be traced. Property is provided during context creation and never propagates outside of process.
`CollectionMode` is enum that has following possible values:

- `ON` - default. Instrumentation is fully enabled.
- `OFF` - Instrumentation is fully disabled. Tracing, propagation or metric updates in scope of this context are fully disabled.
- **[future scenario]** `METRICS` - metrics are enabled
- **[future scenario]** `PROPAGATION` - propagation of context is enabled (from current span & distributed context)
- **[future scenario]** `TRACING` - tracing is enabled
  
Values can be combined together e.g. `ON` translates into `METRICS | PROPAGATION | TRACING`.
DistributedContext inherits `CollectionMode` from the parent/current context when created unless is explicitly changed.

### SDK Behavior

#### CollectionMode = ON

No changes: traces, metrics are collected, contexts are propagated.

#### CollectionMode = OFF

- new nested spans are noop. (if TRACING is not enabled via the bitmask)
- counter add/set/record calls are ignored
- new nested distributed contexts inherit `CollectionMode = OFF` unless overridden by user and are noop
- Propagators do not propagate distributed context and span context.

#### [Future] CollectionMode = METRICS

- metrics are collected as usual.
- new nested spans are noop. (if TRACING is not enabled via the bitmask)
- new nested distributed contexts inherit `CollectionMode` unless overridden by user.
- Propagators do not propagate distributed context and span context. (if PROPAGATION is not enabled via the bitmask)

#### [Future] CollectionMode = TRACING

- counter add/set/record calls are ignored. (if METRICS is not enabled via the bitmask)
- spans are created as usual
- new nested distributed contexts inherit `CollectionMode` unless overriden by user.
- Propagators do not propagate distributed context and span context. (if PROPAGATION is not enabled via the bitmask)

#### [Future] CollectionMode = PROPAGATION

- counter add/set/record calls are ignored. (if METRICS is not enabled via the bitmask)
- new nested spans are noop. (if TRACING is not enabled via the bitmask)
- nested distributed contexts inherit `PROPAGATION` unless overriden by user.
- Propagators propagate distributed context and span context when they are available

Useful combinations (except ON):

- `METRICS` | `PROPAGATION` translates into collecting metrics and propagating distributed context and current span context (if any).
- `TRACING` | `PROPAGATION` translates into tracing and propagating current distributed context (if any) and current span context.

### Adapters behavior

Adapters SHOULD collect metrics, traces and propagation depending on the `CollectionMode` as an optimization only. They MAY also rely on the SDK to turn off tracing/metrics.

### Alternative

The `CollectionMode` could be exposed on the spans but this would imply that spans have to be created even in cases when there is no need for it (scenarios #2 - not tracing internal operations and #3 - not tracing certain endpoints).
