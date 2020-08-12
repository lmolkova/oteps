# Associating sampling priority with the trace using tracestate

Propagating `sampling.priority` (calculated by probability sampler) in a
`tracestate` will enable a better experience for system with the independent
stateless configuration of components sampling.

## Motivation

Consistent sampling decision made in each app of a distributed trace is
important for better user experience of trace analysis. Consistency is achieved
by following means:

- Same hashing algorithms used across all apps in a trace.
- Sampling flag propagated from the head component/app is used by downstream apps to sample in a given trace.

Increasingly more apps participate in distributed tracing. With the
standardized wire format for context propagation, there is bigger chance that
algorithm for sampling chosen by one app may not match another app.
Since coordination of sampling algorithms across many apps not always possible,
OpenTelemetry must provide a way to share sampling hints between apps.

Beyond sampling hints infrastructure, it is beneficial for the community to
agree on standard behavior and hints exposed by probability sampler.

The `sampling.priority` property is used by many vendors for various purposes.
Propagating this field alongside the trace will allow for many improvements and
as a very minimum will simplify transition of customers from SDKs using
different `trace-id` based hash functions to OpenTelemetry SDK.

Also see related discussions on probability sampler here: <https://github.com/open-telemetry/opentelemetry-specification/pull/570>

## Explanation

### Sampling hints exchange

OpenTelemetry SDK has an infrastructure to expose sampling hints as span attributes today
via
[SamplingResult](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#shouldsample)
class.

This recommendation suggests OpenTelemetry will allow to return the
`sampling.priority` that will be appended to the tracestate for a newly created `Span`.
This will allow samplers from a single vendor coordinate sampling decision across
the different components of the trace.

### Probability Sampler default behavior

The default behavior of probability sampler is to calculate a stateless hash
function of `trace-id` to make a sampling decision.

It is suggested to extend the sampler behavior to expose a `tracestate` field
called `sampling.priority` with the floating point value `[0-1]` that will indicate
sampling priority of the current span. The sampling.priority is the probability as calculated
by the hashing function.

This will allow to align sampling algorithms between various components.
Especially for the transition scenarios where components are using different
versions of the SDKs with different sampling algorithms. The components would
respect the sampling.priority if passed in to make sampling decision and not re-calculate
the sampling probability. So let's assume head component A started the trace and
sampled in 60% of the traces and then next component B has sampling policy of
30% but is using different sampling algorithm. In this scenario how can we make sure
that the 30% of the traces sampled in by component B are subset of the 60% traces
sampled in by component A. So, by passing the sampling priority component B can use
the passed in value to make the decision instead of re-calculating with possibly a
different probability algorithm.

## Internal details

Options on `sampling.priority` default behavior. Exposing `sampling.priority` is
not always needed. It also may be desired to not respect incoming
`sampling.priority`. So `ProbabilitySampler` may be configured out of the box to
respect incoming `sampling.priority`. And inserting it into `tracestate` when
not present. A setting can be exposed to NOT write `sampling.priority` to
`tracestate`.

## Trade-offs and mitigations

## Prior art and alternatives

<https://github.com/open-telemetry/opentelemetry-collector/blob/60b03d0d2d503351501291b30836d2126487a741/processor/samplingprocessor/probabilisticsamplerprocessor/testdata/config.yaml#L10>

## Open questions

## Future possibilities
