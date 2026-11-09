# 0134: Pending steps

- Issue: [#134](https://github.com/itinera-dev/spec/issues/134)
- Tier: 2
- Amends: [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0062](0062-workflow-descriptors.md), [0106](0106-retry-budget-alone.md) and [0108](0108-test-support.md)

## Summary

A step may report that it is **pending**: its work is not finished yet. The journey carries on with the steps after it, and when only pending steps remain, the executor tries them again, choosing which first by when each may be tried again (**nbf**, "not before") and by priority (**weight**). A pending step may wait for other pending steps declared before it. A failure ends the journey at once, abandoning the steps still pending. Steps that may be pending, and the workflows that allow them, are declared, and every such step has a budget, so every journey ends. The executor reads time from a clock it is given, and records every choice it makes, with its reasons.

Part of one set with proposals 0133 (hooks before and after a journey, a step and an attempt), whose terms it uses, 0130 (cancellation) and 0135 (timeouts). References such as "#134" are to the issues of those proposals; numbered references such as "6.1 point 4" are to the chapters of specification 0.1.

## Motivation

1. **Work that finishes elsewhere.** A payment being processed, a document awaiting a signature, a job on another system: the honest fact is that the work is not finished yet. Today a step can only fail, which spends the retry budget and stops the journey from doing its other work in the meantime, or wait inside the step, which hides a flow decision in business code.
2. **The basis for other executors.** A pending step is the first time a journey waits with no step running. That is the moment a durable executor stores a journey and a distributed executor hands work to a queue. Defining what pending means now, with the local executor, which simply waits, lets those executors add capability later without changing meaning.

## Design

### Declarations

1. **A step that may be pending is declared so** on its step descriptor (`may be pending`). Pending becomes one of its outcomes, beside Success, Failure and Skipped. A step not declared so that reports pending aborts the journey with the new abort reason `pending not declared`.
2. **A rescue step cannot be pending.** It cannot be declared `may be pending`, and one that reports pending aborts the journey with `pending not declared`. A step with a rescue step may be pending; a pending attempt is not a failure, so it is not rescued.
3. **A workflow allows pending steps** by declaring so on its workflow descriptor (`allows pending steps`). A step that may be pending in a workflow that does not allow them is the new violation `pending not allowed`, reported for every such step. Steps are reusable and workflows assemble them, so a workflow's author consents to waiting explicitly.
4. **Waits.** A step that may be pending MAY name steps it waits for. Each MUST be a step that may be pending, declared before it. Naming an unknown step, a step that cannot be pending, itself or a later step is the new violation `invalid pending wait`. A cycle, and so a deadlock, cannot be expressed.
5. **Settings,** on the step descriptor of a step that may be pending:
   1. a **pending budget**, a whole number of at least 1: how many times the step's pendings may be deferred. A step that may be pending with no budget, or a budget of 0, is the new violation `pending budget missing`;
   2. a **pending deadline**, optional: a duration, counted from the step's first `step_pending`, after which it is not deferred again;
   3. a default **nbf**, a delay counted from the step's `step_pending`, never negative, 0 by default;
   4. a default **weight**, a whole number, 0 by default; a greater weight is tried sooner.

   A negative default nbf, a deadline of 0 or less, or any of these settings declared on a step that cannot be pending, or waits declared by such a step, is the new violation `invalid pending setting`.
6. **When they are found.** The four violations need only the declaration, so they are among the errors caught when the workflow is built (3.6 point 1, 3.7).
7. **The listing** shows `allows pending steps`, and for each step that may be pending its waits and settings.

### Data

What a step that may be pending contributes, and who reads it, is the developer's responsibility: a step that cannot be pending MUST NOT depend on data from one that can, since it may run before that data exists. The executor cannot check this, as contributions are not declared; a mistake meets the usual rules, `required data missing` or an optional input absent. A pending attempt's contributions are discarded, as a skipped attempt's are, with `contributions_discarded`.

### The clock

1. **The executor is given a clock** when it is created, the system's clock by default. It reads every time it uses from that clock: nbf, deadlines, how long it waits, and the timestamp of every event (2.5).
2. **Conformance runners drive the clock.** A runner gives the executor a clock it controls. When the executor waits, the runner's clock moves to the moment the executor waits for, so no case waits in real time, and the cases check nbf, deadlines and order exactly.
3. **Test support** (chapter 8) offers such a clock to applications, as a convenience: it moves only when told, or to the moment the executor waits for.
4. **Durations are whole milliseconds.** Every duration in this proposal, the pending deadline and every nbf delay, is a whole number of milliseconds, and the executor reads and compares time in whole milliseconds. Each language accepts its own standard form, such as a duration type or an object naming days, hours, minutes, seconds and milliseconds, and converts it exactly; a duration with a fraction of a millisecond is refused, never rounded: at declaration as `invalid pending setting`, and from `on step pending` as an `unexpected result`. Events carry durations as whole milliseconds, and times on the clock as ISO 8601 in UTC, as timestamps are (2.5). How each language spells a duration belongs to its tech spec.

### Passes

A journey runs in **passes**, each named by what decides its order:

1. **The ordered pass:** the scan, as in tier 1. Every step reached in declared order gets its first run there, its retries and rescue included, and nbf and weight play no part in it: a pending step only lets the scan move on. The one exception is a step whose turn comes while a step it waits for is not done (point 4 of "The ordered pass"): it becomes `Waiting`, and its first attempt comes in the scheduled pass.
2. **The scheduled pass:** it starts only when the ordered pass ends with every step `Succeeded`, `Skipped`, `Bypassed`, `Pending` or `Waiting`, and at least one step `Pending` or `Waiting`. The executor schedules which of those steps to try again, by the rule of "Choosing which step to try next". A journey with no pending step never enters it.
3. **`pass_started`,** a new fact, is emitted once when each pass starts: for the ordered pass after `journey_started`, once every `before journey` hook (#133) has continued, so a journey that fails, is aborted or is cancelled there has no `pass_started`, and for the scheduled pass when the ordered pass ends with steps left. It carries the pass, `ordered` or `scheduled`, and for the scheduled pass the steps still `Pending` or `Waiting`. The set of passes is closed in this tier, and a later one may add a pass without a new event.
4. Chapter 1 gains the terms pass, ordered pass and scheduled pass.

### The ordered pass

1. Every step's first attempt still comes in declared order, as today, except for a step that waits (point 4).
2. **An attempt that reports pending** ends with the new outcome fact `step_pending`. Then, in order: the attempt's `after attempt` hooks (#133), its contributions discarded, and exactly one step decision, `step_deferred` or `step_given_up`, by the rules of "The budget" below. A deferred step becomes `Pending`, and the scan moves on to the next step.
3. **An awaited step is done** when it is `Succeeded`, `Skipped` or `Bypassed` (#133). An awaited step that fails ends the journey, so a waiting step never waits for a step that cannot finish.
4. **A step whose turn comes while a step it waits for is not done** is not attempted: its `before step` hooks run (#133), then `step_waiting`, it becomes `Waiting`, and the scan moves on.
5. **When the scan reaches the end** and only `Pending` and `Waiting` steps remain, the scheduled pass starts: the executor chooses, again and again, which to try next, until none remains or the journey ends.
6. **Once chosen, a step runs as in the scan:** its retries after a failure happen at once (5.6 point 5), and its rescue as chapter 7 says, until it succeeds, skips, is given up or reports pending again.
7. **The journey succeeds** when no step is left that is not `Succeeded`, `Skipped` or `Bypassed`.

This amends 5.5 points 1 and 2: the scan passes over `Pending` and `Waiting` steps, and the scheduled pass is the one exception to strict order.

### Choosing which step to try next

1. **Eligible:** a `Pending` step whose nbf has passed and whose awaited steps are all done, or a `Waiting` step whose awaited steps are all done.
2. **Effective weight:** the greatest of the step's own weight and the weights of every step that waits for it, directly or through others. A group of dependent steps is judged by its heaviest member against independent steps, and a heavy step is never starved behind the light step it needs.
3. **Rounds.** A step's **round** is how many times it has been chosen so far, 0 for a step that was never chosen. Every eligible step of a lower round is chosen before any of a higher round, so every eligible step is tried once before any is tried again.
4. **Order among the eligible:** lower round first, then greater effective weight, then earlier nbf, with a `Waiting` step's nbf being the moment it became eligible, then declared order.
5. **nbf is never overridden by weight or round:** a step is never chosen before its nbf. The executor never waits while a step is eligible.
6. **None eligible:** the executor waits on its clock until the earliest moment a step becomes eligible, or until the journey timeout passes (#135), whichever comes first. A synchronous executor blocks, and an asynchronous one awaits; the meaning is the same in both modes.

### The budget, the deadline and `on step pending`

This mirrors the retry budget (5.6, 6.2):

1. **Budget left:** with a pending budget of N, the first N pendings of a step are each deferred. `on step pending` is called, then `step_deferred`.
2. **Budget spent:** on the pending after the Nth, `on step pending` MUST NOT be called, since no deferral is possible. `step_given_up` follows with the new cause `pending exhausted`, saying `budget`, and then only `on step failure`.
3. **The deadline:** when the nbf decided for a pending, by default or by `on step pending`, falls after the step's deadline, the step is not deferred: `step_given_up` follows with the cause `pending exhausted`, saying `deadline`, decided by `default`, and then only `on step failure`. The deadline is checked only here, so a step chosen after its deadline because others ran first is still tried.
4. **New step hook `on step pending`,** defined at most once per step like today's outcome hooks. It MAY request the step's name, the attempt number, the journey ID, the step's hint, how many deferrals are left, data from the workflow and roles, as today's step hooks may (6.6).
5. **It returns** nothing, which keeps the defaults; **defer**, with a new nbf delay, a new weight or both, for this pending only; or `FailWorkflow`. Defer is not a lifecycle: it acts on one step. `FinishWorkflow` is an `invalid lifecycle`. A negative delay SHOULD be impossible to express; otherwise it aborts the journey with `unexpected result`.
6. **`FailWorkflow` from `on step pending`:** the step becomes `Failed`, `step_given_up` is emitted with the cause `FailWorkflow`, decided by that policy and hook, and `on step failure` MUST NOT be called, as for `on step retry` (6.3 point 4).
7. **The hint.** A step reporting pending MAY attach a hint, a value (4.6), such as the "retry after" an external system gave it. The step suggests; the workflow decides. A hint that is not a value aborts the journey with `not a value`.
8. **nbf is fixed once decided:** from the default or the hook, a delay from the step's `step_pending`, recorded by `step_deferred`. Only a new pending sets a new one.
9. **Attempts:** trying a pending step again starts a new attempt, with `attempt_started` and the next attempt number. Pendings do not spend the retry budget, which stays for failures; a failure on a later attempt follows today's retry and rescue rules. A step may so have up to its pending budget more attempts than 5.6 point 1 allows, which it amends.

The 6.2 list and the 6.3 table gain these rows.

### Ending while steps are pending

1. **A failure** (a step given up, or `FailWorkflow` from any hook) and **`FinishWorkflow`** end the journey at once, as today. Then, in order:
   1. the step that ended the journey finishes as today and as #133 says, its `after step` hooks included, up to the last hook it calls: `on step failure`, or the hook that returned `FailWorkflow` or `FinishWorkflow`;
   2. then, after the ending step's last event, which is the `hook_called` of `on step failure` when it is called, of the hook that returned `FinishWorkflow` from `on step success`, or otherwise the step's `after step` hooks, for each `Pending` or `Waiting` step, in reverse order of when its turn began (#133, point 5 of "The hooks"): `step_abandoned`, decided by the hook that ended the journey or by `default`, then its `after step` hooks, with the ending `abandoned`; their failures are recorded and change nothing (#133);
   3. then the journey's end, as #133 says: `after journey`, the workflow hook, then `journey_failed` or `journey_succeeded`.
2. Steps that never had their turn stay `NotExecuted` (5.4 point 2, 6.3 point 3).
3. **An abort** ends the journey at once, as today. No `step_abandoned` is emitted, since `journey_aborted` is the last event (2.4 point 3). A `Pending` or `Waiting` step becomes `Aborted`, as a `MustRetry` step does (5.4), and its `after step` hooks run as #133's "After an abort" says.
4. The statuses and endings follow the table shared by #133, #134 and #130 (#133, "Statuses and endings").

### Test support

1. **The path** (8.3) gains the lines `scheduled pass started`, `<step> attempt <n> pending`, `<step> deferred`, `<step> waiting`, `<step> selected`, `<step> abandoned`, and the causes `pending exhausted: budget` and `pending exhausted: deadline` for `<step> given up`. `pass_started` for the ordered pass has no line, so the paths of journeys with no pending step stay as in tier 1. 8.3 point 2 is amended, since `pass_started` is neither an outcome nor a decision.
2. **Scripted runs** (8.5) gain the endings `will be pending` and `will be pending with hint` a value, and the attempts a step can have (8.5 points 7.1 and 8.2) count its pending budget. `will be pending` for a rescue step, or for a step not declared `may be pending`, is a mistake in the test (8.5 point 8), as a rescue step that skips is.
3. **The clock** of "The clock" point 3.

### Relation to other rules

1. **5.6 point 5** ("A retry happens immediately. The executor MUST NOT wait between attempts.") holds for retries after a failure. Trying a pending step again is not a retry, and waits for its nbf.
2. **4.8** stays true: the executor never interrupts a running step. It waits only between attempts, when no step runs.
3. **VISION principle 3,** "order is strict and visible", is reworded, and the order stays guaranteed in the ordered pass. The scan still runs every step once in declared order, retries and rescues included, and nbf and weight play no part in it: a pending step only lets the scan move on. Only steps that reported pending, and steps that declared they wait for one, are run again or for the first time in the scheduled pass, by the declared rule of "Choosing which step to try next", and the stream records every such choice with `step_selected`. The new wording: "Order is strict and visible. Every step gets its first run in the order the workflow declares, retries included, unless it declares that it waits for a step still pending. Only steps that report pending, and those waiting for them, run after that pass, by a declared rule, and the event stream records each choice."
4. **#133:** `after attempt` runs after each pending attempt, so nothing opened for an attempt is held across a wait. `before step` runs once, at the step's first turn; `after step` runs when the step reaches its final status, which for a step that may be pending may come after later steps have run.
5. **#130:** a wait is a boundary at which cancellation stops the journey.

### Implementation ideas

The local executor keeps the pending and waiting steps in a structure ordered by the choosing rule, and the clock is an interface given to the executor. A synchronous executor sleeps the thread until the next eligible moment; an asynchronous one awaits a timer. How each language waits, and how its clock is given and driven, belongs to each language's tech spec.

## Events

Every choice the executor makes is a decision, with what it was based on. Every time an event carries comes from the executor's clock.

| Event | Kind | When | Carries |
|---|---|---|---|
| `step_pending` | fact | an attempt reported that its work is not finished yet | step, attempt, the hint if any |
| `step_deferred` | decision | a pending is deferred, after `on step pending` | step, attempt, nbf (as a delay in milliseconds and as a time on the executor's clock), weight, deferrals left, deadline if any, decided by |
| `step_waiting` | decision | a step's turn came while a step it waits for is not done | step, the steps it waits for that are not done, decided by `default` |
| `step_selected` | decision | the executor chose the next step to try in the scheduled pass | step, the attempt it will start, its status before (`Pending` or `Waiting`), its round, its effective weight and, when lifted, the step it took it from, its nbf, the other eligible steps it was chosen over, decided by `default` |
| `pass_started` | fact | a pass starts | the pass (`ordered` or `scheduled`); for the scheduled pass, the steps still `Pending` or `Waiting` |
| `journey_waiting` | fact | no step is eligible, and the executor waits | until when, and the step that becomes eligible then, or none when the journey timeout (#135) comes first |
| `step_abandoned` | decision | the journey ended by a failure or `FinishWorkflow` while the step was `Pending` or `Waiting` | step, its status before, the journey's ending, decided by |

1. **Outcome facts.** `step_pending` joins the outcome facts of 2.7 point 1, so each attempt still has exactly one. 2.7 point 2 gains: exactly one step decision, `step_deferred` or `step_given_up`, MUST follow every attempt that ended in `step_pending`.
2. **The hint is carried.** `step_pending` carries the hint's value, an exception to 2.6 point 4 like the data of `service_provided`: the step wrote it for the workflow to read.
3. **New statuses** in 5.4 and the 5.5 diagram: `Pending`, from `Executing`; `Waiting`, from `NotExecuted`; and `Abandoned`, final, from `Pending` or `Waiting`, and on a cancellation from the statuses #130 names. `Pending` and `Waiting` go to `Executing` when the step is chosen, and to `Aborted` on an abort. There is no transition from `Pending` to `Waiting`: a step that reported pending has had its turn.
4. **New cause** `pending exhausted` on `step_given_up`, saying `budget` or `deadline`. The 5.8 table gains its row: no reason and no error.
5. **New abort reason** `pending not declared`; **new violations** `pending not allowed`, `invalid pending wait`, `pending budget missing` and `invalid pending setting` in 3.6. A step not declared `may be pending` that reports pending, a rescue step that does, and a negative delay from `on step pending` are rules a language SHOULD make impossible to express, listed in 3.8 with their tags.
6. `hook_called` records `on step pending` and what it returned.
7. `journey_waiting` is the moment a durable executor will store a journey; this proposal gives it no further meaning.
8. **Events with no attempt.** `step_waiting` and `step_abandoned` are about a step but no attempt, and `step_selected` carries the attempt it will start; 2.5 point 2 gains this form, as for #133's step-level events.
9. **Order** (2.8): `pass_started` for the ordered pass comes after the `before journey` hooks and before the first step's turn, and for the scheduled pass after the ordered pass's last event and before the first `journey_waiting` or `step_selected`; `step_pending` is the attempt's outcome fact, followed by its `after attempt` hooks, `contributions_discarded`, `on step pending` and its decision; `journey_waiting` comes before the `step_selected` that follows the wait; `step_selected` comes before the chosen step's `attempt_started`.
10. **Also amended:** 1.2 and 4.3 point 1, where pending becomes a fourth outcome, and chapter 1's list of statuses; 5.7 points 1 to 3, for how a journey with pending steps ends; 3.1 point 2 and 3.2, for the new declarations on the workflow and step descriptors; 8.1 point 1, since the clock joins what test support offers.

## Conformance

Cases, tagged with this proposal and `@tier-2`, with the runner driving the executor's clock. They check:

1. the ordered pass moving past a pending step, the scheduled pass returning to it, and `pass_started` for each, with what it carries;
2. nbf respected, and never overridden by weight or round;
3. the order by round, effective weight, nbf and declared order, including a light step lifted by a heavy step that waits for it;
4. rounds: a light step is tried before a heavy step is tried a second time;
5. waits: `Waiting`, its release when the awaited step succeeds, is skipped or is bypassed, and `before step` before `step_waiting`;
6. `on step pending` deferring with a new nbf and weight, returning `FailWorkflow`, and not being called when the budget is spent;
7. the budget and the deadline giving a step up with `pending exhausted`, then `on step failure`;
8. pendings not spending the retry budget, and a failure after a pending retried and rescued as today;
9. a pending attempt's contributions discarded;
10. abandonment on a failure and on `FinishWorkflow`, in the order of "Ending while steps are pending", with `after step` given the ending `abandoned`;
11. an abort while steps are pending: no `step_abandoned`, and their `after step` hooks run;
12. the hint carried by `step_pending` and given to `on step pending`, and a hint that is not a value;
13. a rescue step that reports pending, and a step not declared `may be pending` that does;
14. every new violation, including `invalid pending setting`, and the listing;
15. the events, their contents, and timestamps from the executor's clock;
16. the new path lines and scripted endings.

## Alternatives considered

1. **Pending blocks the journey** until the step succeeds: keeps the order strict, but loses doing other work while waiting.
2. **Steps declare the keys they contribute,** and later steps that request them wait: safe automatically, but a much larger change to every step; the developer's responsibility and declared waits serve the same need.
3. **nbf and weight decided by the step:** a flow decision inside business code; the step gives a hint, and the workflow decides.
4. **A workflow-wide pending limit or deadline:** not needed for every journey to end, since each step's budget bounds it; a journey deadline can come later, checked between steps.
5. **A capability for pending in synchronous executors only:** the meaning is the same in both modes, and only how the executor waits differs.
6. **Weight before rounds:** a heavy step that keeps reporting pending would starve every lighter step; rounds keep weight as a priority within each round.

## Open questions

None.
