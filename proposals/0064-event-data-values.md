# 0064: Event data and reason details are values, serialized only by reporters

- Issue: [#64](https://github.com/itinera-dev/spec/issues/64)
- Tier: 1
- Amends: [0008](0008-steps.md), [0011](0011-events.md)

## Summary

The data a step or hook attaches to its own events (`step_info`, `journey_warning` and the rest) is carried in the event as a value, in memory, and is not serialized by the executor. It must be serializable, because reporters may want to serialize it, but serialization happens only in a reporter, in whatever format that reporter chooses. The same holds for the details of a reason.

Amends proposal 0011 (item 22, spec 2.9 point 3: "The data MUST be serializable and is serialized when the event is emitted; if it cannot be serialized, the event MUST still be delivered, with a marker saying so"), and the "details" of reasons in 4.3 point 3.

## Motivation

Serializing at emission picks one format for every reporter. JSON is the natural choice, but a reporter may write a binary format, forward to a tracing system with its own encoding, or keep the value in memory, as a test's recording reporter does. Serializing first means:

- a cost paid on every emitted event, even when every reporter ignores the data;
- a format chosen by the executor, which a reporter may have to parse back;
- typed values turned into text before any reporter can look at them.

What the rule protects can be kept without it:

- data is captured when emitted, because the event carries its own copy of the value, so changing the original afterwards changes nothing (as for contributions, 4.6 point 3);
- data is always serializable, because it must be a value, in the sense of #56.

## Design

1. **Data in step and hook events is a value.** The data a step or hook attaches to its events MUST be a value: data that can be serialized, as #56 defines. Functions, closures, handles and other non-values are not allowed.
2. **Captured, not serialized.** The event carries its own copy of the value, captured when the event is emitted: changing the original afterwards MUST NOT change the event. The executor MUST NOT serialize it.
3. **Serialization belongs to reporters.** A reporter MAY serialize the data, in any format. If it fails to, that is the reporter's business, under the rules for reporters (2.3).
4. **Data that is not a value is illegal.** Where a language can express it, it is detected when the event is emitted: the event is not delivered, the journey is aborted with `not a value` (#56), naming the step, or the policy and hook, and the event kind, and the emit call ends the step or hook with the executor's signal, as #55 defines. The marker for data that cannot be serialized is removed. Implementations SHOULD make it impossible to express (#54), under the tag `@non-value`.
5. **Reasons.** The optional details of a reason (4.3 point 3) follow the same rules: a value, captured when the reason is made, never serialized by the executor. A step or hook whose reason carries details that are not a value aborts the journey with `not a value`.
6. **Unchanged:** engine events never carry values from the data bag (2.6 point 4). This proposal concerns only the data a step or hook chooses to attach to its own events, and the details of reasons.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0064-event-data-values/`, with the amended scenarios of earlier proposals tagged `@proposal-0064`.

## Alternatives considered

- **Serializing at emission** (the current specification). Rejected: it fixes one format for every reporter and costs on every event.
- **No constraint on event data.** Rejected: reporters must be able to serialize what they receive, and non-values would pass behaviour through the event stream.

## Open questions

None.
