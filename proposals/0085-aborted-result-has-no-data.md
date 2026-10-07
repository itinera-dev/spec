# 0085: An aborted journey's result carries no data bag, and an abort is final

- Issue: [#85](https://github.com/itinera-dev/spec/issues/85)
- Tier: 1
- Amends: [0065](0065-business-result.md) and [0009](0009-running-a-workflow.md)

## Summary

1. An aborted journey's result carries the journey ID and the abort (its reason, its details, and its error when custom code caused it), but no data bag.
2. A succeeded or failed journey's result keeps the data bag, unchanged.
3. An abort is final: it means something illegal happened, typically code that did not handle an error as it should. An aborted journey has no business outcome and can never be resumed, in this tier or any later one.
4. What happens to contributions still being committed when the journey is aborted then becomes unobservable, which closes a gap in the specification.

Amends proposal 0065 (the result) and proposal 0009 (item 17, the data bag is the journey's output). In the chapters it changes 1.3 (the data bag), 1.5 (aborted journey), 5.7 and 5.8.

## Motivation

Spec 5.8 point 1.3 requires every result to carry the data bag as it stood at the end, aborted results included. An abort is not a business outcome: it is a fault, a configuration error or failing custom code (5.7, 5.10), and the journey stops wherever it was. A caller who reads the data bag of an aborted journey may act on data that stands halfway through a change, and the specification cannot promise what it holds.

There is a concrete case. After a successful attempt, the executor commits the step's contributions one key at a time, each followed by `contribution_committed` (2.8 point 3), and a hook's contributions the same way (2.8 point 4). If a reporter fails on one of those `contribution_committed` events, the journey is aborted with `reporter failed` (2.3). The specification does not say whether the remaining keys of that success are still committed, so the data bag in the result may hold only part of what the step contributed.

Nothing runs after an abort: no step, no hook, and no event except `journey_aborted`. The result is therefore the only place where that partial state could be seen. Taking the data bag out of an aborted result removes both the risk and the gap, without having to define partial commits.

Whoever needs to know what happened before the abort reads the event stream: every committed key is recorded by `contribution_committed`, and why the journey was aborted by `journey_aborted`. Engine events carry keys only, never values (2.6 point 4), so after an abort the values of the data bag are not observable anywhere. That is accepted: an abort has no business data. A workflow that wants values in its diagnostics emits them itself, as the data of its own `step_*` or `journey_*` events.

## Design

1. **An aborted result carries no data bag.** The result of an aborted journey MUST carry the journey ID and the abort: the abort reason, its details, and the error when custom code caused the abort by failing (5.8 and 5.10). It MUST NOT carry the data bag, in whole or in part.
2. **Succeeded and failed results are unchanged.** They MUST carry the data bag as it stood at the end: the initial data plus every committed contribution.
3. **An abort is final.** An abort means something illegal happened, typically code that did not handle an error as it should. An aborted journey has no business outcome, and it MUST NOT be resumed or continued, by any executor, in this tier or any later one.
4. **The journey's output.** The data bag is the journey's output only when the journey succeeded or failed. The definition of the data bag in 1.3 becomes: "At the end of a journey that succeeded or failed, it is the journey's output." The definition of an aborted journey in 1.5 gains: "An aborted journey has no output and is never resumed."
5. **Commits interrupted by an abort.** When a journey is aborted while the contributions of a successful attempt or of a hook are being committed, whether the remaining contributions are committed is not observable and is left to the implementation. No `contribution_committed` is delivered after the abort, as 2.3 already requires.
6. **The text of 5.8 point 1.3** becomes: "for a journey that succeeded or failed, the **data bag** as it stood at the end: the initial data plus every committed contribution. The data bag is the journey's output. An aborted journey's result carries no data bag: an abort is a fault, and the data bag may stand halfway through a change." 5.7 point 3 gains point 3 of this design.
7. **Unchanged.** Refusals carry no data bag, as today. The event stream is unchanged.

## Events

No new events. The event stream is unchanged.

## Conformance

The conformance cases for this proposal must check that:

1. an aborted journey's result carries no data bag, and carries the error when custom code caused the abort;
2. the scenarios that looked for a missing key in an aborted result check instead that no contribution of that key was committed.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), tagged `@proposal-0085`, in `cases/tier-1/0009-running-a-workflow/result.feature`. STEPS.md defines the sentences "Then the result carries no data bag" and "Then no contribution of "<key>" was committed".

## Alternatives considered

- **Keep the data bag in every result** (the current specification), documented as possibly incomplete after an abort. A caller would still have to know not to trust it, and the specification would still need to define what an interrupted commit leaves behind.
- **Define interrupted commits instead**, for example that a step's or hook's contributions are committed all at once, before any `contribution_committed` is delivered. That fixes the partial state but not the purpose: an aborted journey has no business output, and the data bag would still invite callers to act on it.
- **Keep only the initial data in an aborted result.** It is already in the caller's hands, since the caller supplied it with the workflow instance, so it adds nothing.
- **Leave the data bag of an aborted journey available to later tiers**, for example so that a durable executor could resume it. Rejected: an abort is final and has no business data, in every tier.

## Open questions

None.
