# 0055: A reporter that fails while a step or hook runs

- Issue: [#55](https://github.com/itinera-dev/spec/issues/55)
- Tier: 1
- Amends: [0011](0011-events.md)

## Summary

Define when the events a step or hook emits are delivered, and what happens when a reporter throws while the step or hook is still running.

- Delivery is blocking: an event is delivered to every reporter before the emit call returns.
- A reporter that throws during that delivery aborts the journey with `reporter failed`, and the emit call ends the running step or hook. The abort is recorded first, so it stands even if the step or hook catches whatever ended it.

Amends proposal 0011 (spec 2.9 point 4, "Emitting an event never fails and never changes an outcome"), and the reporter row of the 5.10 table from proposal 0049.

## Motivation

Steps emit `step_info`, `step_warning` and `step_error`, and hooks emit `journey_*` events, while they run. The specification does not say when those events are delivered. It also does not say what happens when a reporter throws on one of them while the step or hook is still running.

- 4.8 forbids the executor to interrupt a running step.
- 2.3 says a reporter that throws aborts the journey.
- 2.4 point 3 says `journey_aborted` MUST be the last event.

So, between the throw and the moment the step returns, it is undefined whether the step keeps running, whether its later events are delivered, whether its outcome fact is emitted, and what its status becomes. A step that keeps running behind an aborted journey can still cause side effects, such as a payment, after the journey is already aborted.

Buffering a step's events and delivering them when it returns was considered and rejected: if the step ended in an unrecoverable failure, such as a crash of the process, the buffered events would be lost, and they are exactly the events needed to diagnose it.

## Design

1. **Blocking delivery.** An event a step or hook emits MUST be delivered before its emit call returns: the call blocks for a synchronous step or hook and is awaited for an asynchronous one. Delivered means every reporter's `report` operation has returned. What a reporter does with the event afterwards, such as putting it in a queue, is its own business and outside the journey.
2. **A reporter that throws while a step or hook runs.** The executor wraps every call to a reporter. If a reporter throws during the delivery of an event emitted by a running step or hook:
   1. the journey is aborted with `reporter failed`, and the abort is recorded at once;
   2. the event it threw on is still delivered to the reporters after it (2.3 point 2);
   3. the emit call then ends the step or hook, in the way the language allows: by throwing an exception that belongs to the executor, or, in a language without exceptions, by returning the executor's error, which the step or hook propagates.
3. **The abort stands whatever the step or hook does.** If the step or hook catches what ended it and carries on, the journey is still aborted. Nothing more it emits is delivered, and when it returns, its outcome, its returned lifecycle and its contributions are ignored: no outcome fact, no `hook_called`, no commit.
4. **It is not an abnormal termination.** What ends the step escaping it MUST NOT be treated as an abnormal termination: no `step_abnormal_termination`, no retry and no failure hook. For a hook, the abort reason is `reporter failed`, not `hook failed`.
5. **Then `journey_aborted`.** When the step or hook returns, `journey_aborted` is delivered with the reason `reporter failed`, as 2.3 point 2 defines, naming the step, whose status becomes `Aborted`. For a step hook, the step is the one the hook acts on.
6. **Implementations SHOULD make what ends the step impossible to catch by name** where their language allows, for example by not exporting the exception type. Point 3 still applies, because a catch-all can catch it.
7. **2.9 point 4 becomes:** "Emitting an event never fails because of the event itself. It ends the step or hook only when a reporter threw while it was being delivered (point 2)."
8. **The executor never interrupts custom code unless it did something illegal.** Spec 4.8 ("the executor never interrupts a running step") becomes the general principle: the executor never interrupts a step's execution, at any stage (building it, running it, its events, or the hooks after it), as long as the custom code involved does nothing illegal. When custom code does something illegal and the executor learns of it, the executor aborts the journey; where an executor-owned signal is the only way to stop the custom code, as with an emit call after a reporter failed, it ends that code, whatever the code does afterwards is ignored, and the abort stands. Spec 6.5 points to the same principle for hooks.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0055-reporter-fails-while-custom-code-runs/`, with the amended scenarios of earlier proposals tagged `@proposal-0055`.

## Alternatives considered

- **Buffering a step's events and delivering them when it returns.** Rejected: an unrecoverable failure would lose the events emitted before it, and progress would not be visible while the step runs.
- **Letting the step run on, dropping its later events.** Rejected: the step keeps causing side effects in a journey that is already aborted.
- **Relying only on an unexported exception type.** Rejected as the only guarantee: a catch-all still catches it, so the recorded abort (point 3) is what makes the outcome certain.

## Open questions

None.
