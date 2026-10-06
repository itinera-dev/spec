# 0081: A reporter that fails stops the delivery of the event it failed on

- Issue: [#81](https://github.com/itinera-dev/spec/issues/81)
- Tier: 1
- Amends: [0049](0049-custom-code-that-throws.md) and [0055](0055-reporter-fails-while-custom-code-runs.md)

## Summary

Reporters should not fail. A reporter that cannot report an event handles that itself and returns normally, so the reporters after it still receive the event. When a failure does escape a reporter, delivery of that event stops at that reporter: the reporters after it do not receive it. The journey is aborted with `reporter failed`, as before, and `journey_aborted` is delivered to every reporter except the one that failed.

Amends proposal 0049, item 6 (the correction of spec defect #47), and proposal 0055, design point 2.2. In the chapters it changes spec 2.1 point 4, 2.2 point 4, 2.3 point 2 and 2.9 point 5.2.

## Motivation

Reporters should never fail (2.1 point 4, 2.3 point 7). When one does, the journey is aborted, and nothing else in the journey is tried: the specification already emits nothing after the abort except `journey_aborted`. Delivering the event the reporter failed on to the reporters after it was the one exception to that, because it kept working on a journey that was already aborted.

The exception also left a case undefined. If two reporters failed on the same event, 2.3 point 2 said `journey_aborted` goes to every reporter "except the one that failed", and did not say whether the second reporter that failed receives it. If delivery stops at the first failure, only one reporter can fail per event, so the question no longer arises.

What keeps the other reporters receiving every event is the reporters themselves: a reporter that cannot do its work for one event, for example because the service it forwards to is unavailable, deals with that and carries on.

## Design

1. **Reporters handle their own trouble.** A reporter SHOULD NOT fail. A reporter that cannot report an event SHOULD handle that itself, as it chooses (logging it, dropping the event, keeping it to retry later), and return normally, so that the reporters after it still receive the event. A failure that escapes a reporter aborts the journey (2.3).
2. **Delivery stops at the reporter that failed.** When a reporter fails on any event except `journey_aborted`, the event it failed on MUST NOT be delivered to the reporters after it. The journey is aborted with `reporter failed`, and the abort is recorded at once.
3. **Then `journey_aborted`.** `journey_aborted` MUST be delivered to every reporter the dispatcher holds except the one that failed, including the reporters that did not receive the event it failed on.
4. **Whatever the dispatcher.** Points 2 and 3 hold whatever the dispatcher, for the reporters added to it for the journey: once a reporter has failed, none of them receives any further event except `journey_aborted`, and the one that failed receives nothing more, even if the dispatcher would go on delivering.
5. **The same rule whoever emitted the event.** It applies to engine events, and to `step_*` and `journey_*` events emitted by a running step or hook. For the latter, everything else in 2.9 point 5 is unchanged: the emit call ends the step or hook, the abort stands whatever it does afterwards, and `journey_aborted` is delivered when it returns.
6. **Unchanged.** A reporter or dispatcher that fails while `journey_aborted` is being delivered is still ignored, and delivery of `journey_aborted` continues to the remaining reporters (2.3 point 4). A dispatcher that fails while dispatching still aborts with `reporter failed` (2.3 point 3).

## Events

No new events. Which reporters receive the event a reporter failed on changes, as the design says.

## Conformance

The conformance cases for this proposal must check that:

1. the event a reporter failed on does not reach the reporters after it, and every other reporter receives `journey_aborted`, while the one that failed does not;
2. delivery stops at the reporter that failed for an engine event, and for an event a running step emitted.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), in `cases/tier-1/0011-events/reporters-and-dispatchers.feature`, tagged `@proposal-0081`. The scenarios of `0055-reporter-fails-while-custom-code-runs/` list their reporters so that they hold under both rules.

## Alternatives considered

- **Keep the correction of #47**: the event still reaches every reporter after the one that failed. Its benefit is that every reporter that keeps working sees the full stream up to the abort. Not chosen, because it keeps delivering events for a journey that is already aborted, and leaves undefined what happens when two reporters fail on the same event. Reporters that handle their own trouble (point 1) keep the stream complete without it.
- **Keep #47 and define the two-failure case**, by skipping every reporter that failed when `journey_aborted` is delivered. Not chosen, for the first reason above.

## Open questions

None.
