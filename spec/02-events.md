# 2. Events

Specification 0.1. Written from proposal [0011](../proposals/0011-events.md) as amended by [0040](../proposals/0040-events-facts-and-decisions.md) and [0041](../proposals/0041-hook-data-from-the-data-bag.md), [0054](../proposals/0054-how-custom-code-fails.md), [0055](../proposals/0055-reporter-fails-while-custom-code-runs.md), [0063](../proposals/0063-dispatcher-factory.md), [0064](../proposals/0064-event-data-values.md), [0081](../proposals/0081-reporter-failure-stops-delivery.md) and [0083](../proposals/0083-what-failures-carry.md), with the event-stream rules of [0032](../proposals/0032-configuration-errors.md), the rules of [0049](../proposals/0049-custom-code-that-throws.md) for reporters that fail, and the events introduced by [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md) and [0009](../proposals/0009-running-a-workflow.md).

Events come first because every conformance case observes a journey through its event stream. The events about steps, hooks and data are listed here in full; what makes each happen is defined in the chapters that follow.

## 2.1 Reporters

1. A **reporter** has one operation, **report**, which receives one event. It decides on its own what to do with it: filter it, map it, forward it or ignore it.
2. A workflow lists its reporters. Which reporters a journey is reported to is the workflow's decision, never the executor's.
3. A reporter MAY be synchronous or asynchronous, under the executor's execution modes (chapter 5).
4. **Reporters handle their own trouble.** A reporter SHOULD NOT fail. A reporter that cannot report an event SHOULD handle that itself, as it chooses (logging it, dropping the event, keeping it to retry later), and return normally, so that the reporters after it still receive the event. A failure that escapes a reporter aborts the journey (2.3).

## 2.2 The dispatcher

1. An **event dispatcher** has two operations: **add a reporter**, and **dispatch an event** to the reporters it holds.
2. **A dispatcher factory** creates dispatchers. An executor is given a dispatcher factory when it is created, or uses the default factory, which creates the default dispatcher.
3. **One dispatcher per journey.** For each journey, the executor MUST call its factory once, before the journey starts, add the workflow instance's reporters to the dispatcher it returns, and dispatch every event of the journey through it. When the journey ends, the executor drops that dispatcher, with every reporter added to it, so a reporter never receives another journey's events.
4. **The default dispatcher** MUST call reporters in the order they were added, with every event in sequence order, and every reporter it holds MUST receive the same events, except as 2.3 defines when a reporter fails.
5. **Other dispatchers** decide what adding a reporter means and how they deliver; for example, a test's factory may return dispatchers that all hold the test's recording reporter and ignore the workflow's reporters.
6. **With no reporter at all**, events are still produced, and go nowhere.
7. **Calling reporters is all a dispatcher does.** A slow reporter slows the journey; that is the workflow author's choice.

## 2.3 A reporter or dispatcher that fails

Custom code fails by throwing or by returning an error (chapter 5, 5.10). A reporter or a dispatcher cannot report an error and carry on, so its failure aborts the journey.

1. A reporter that fails on any event except `journey_aborted` MUST abort the journey with the abort reason `reporter failed`, whatever the dispatcher.
2. The event it failed on MUST NOT be delivered to the reporters after it. Then `journey_aborted` MUST be delivered to every reporter the dispatcher holds except the one that failed. This holds whatever the dispatcher, for the reporters added to it for the journey: once a reporter has failed, none of them receives any further event except `journey_aborted`, and the one that failed receives nothing more, even if the dispatcher would go on delivering.
3. A dispatcher that fails while dispatching an event other than `journey_aborted` aborts the journey in the same way, with `reporter failed`.
4. A reporter or a dispatcher that fails while `journey_aborted` is being delivered MUST be ignored: the journey is already aborted, its first abort reason stands, no further event is emitted, and delivery continues to the remaining reporters.
5. A dispatcher factory that fails, or a dispatcher that fails while the journey's reporters are added, is a refusal, before any event (chapter 5, 5.10).
6. When a reporter fails on an event a running step or hook emitted, the step or hook is ended at its emit call (2.9 point 5).
7. Reporters, dispatchers and dispatcher factories SHOULD be written so that they never fail.

## 2.4 The event stream

1. The **event stream** of a journey is every event dispatched for it, in order.
2. Events MUST NOT be rolled back or rewritten, even when an attempt's contributions are discarded.
3. An aborted journey keeps every event emitted before the abort, and `journey_aborted` MUST be its last event.
4. A workflow refused before any journey exists, when it is built or handed to the executor, MUST NOT produce any event (chapter 3).
5. **The conformance trace** of a journey is its event stream. A workflow refused before any journey exists has no stream; its trace is the refusal itself, the error listing every violation.

## 2.5 Fixed fields

1. Every event MUST carry:
   - its **kind**, in snake_case;
   - a **sequence number**, increasing within the journey;
   - a **timestamp**, in UTC, in ISO 8601;
   - the **journey ID**;
   - the **workflow name**.
2. An event about a step MUST also carry the **step name** and the **attempt number**.
3. An event from or about a hook MUST also carry the **policy name** and the **hook name**, and, for a step hook, the step name and attempt number that triggered it.
4. `journey_failed` and `journey_aborted` are events about the journey, not about a step. They explain why the journey ended: `journey_failed` names the step that failed, and `journey_aborted` the step during which it was aborted, if any, but neither carries an attempt number.

## 2.6 Facts and decisions

1. Engine events are of two kinds.
   - **A fact** records something that happened, when it happens. It MUST NOT depend on a hook.
   - **A decision** records what the executor decided to do next. It MUST be emitted only once the decision is final, after every hook that could change it has returned, and MUST carry **decided by**: either `default`, the executor's own rule, or the policy and hook whose returned lifecycle decided it.
2. So the stream always reads: what happened, the hooks that were consulted, what was decided.
3. There are no separate events for step status changes: every step status follows from the facts and decisions below.
4. Engine events MUST NOT carry data values, only keys. Reasons and error messages are carried, because their authors wrote them.

## 2.7 The engine catalogue

The executor MUST emit exactly these engine events, and no others, in tier 1.

Facts:

| Event | When | Also carries |
|---|---|---|
| `journey_started` | the journey starts | the initial keys of the data bag, never their values |
| `attempt_started` | an attempt of a step starts | step, attempt |
| `input_adapter_supplied` | an input adapter supplies a value for a step's input | step, key, adapter |
| `input_adapter_failed` | an input adapter fails, or supplies something that is not a value, for a step's input | step, key, adapter |
| `optional_input_absent` | an optional request has no value: a step's input, a hook's data from the workflow or from the step, or an input adapter's data from the workflow | step, key; the policy and hook, or the adapter, when the request is theirs |
| `step_succeeded` | the attempt reported Success | step, attempt |
| `step_failed` | the attempt reported Failure | step, attempt, retriable, reason (code, message, details) |
| `step_skipped` | the attempt reported Skipped | step, attempt, the reason, if any |
| `step_abnormal_termination` | an error escaped the running step | step, attempt, the error's message |
| `hook_called` | a hook returned without throwing | policy, hook, step and attempt if any, the lifecycle returned or none |
| `contribution_committed` | a contribution reached the data bag | key, source: the step, or the policy and hook |
| `contributions_discarded` | a skipped step's contributions were discarded | step, attempt |
| `data_overwritten` | a committed key replaced an earlier value | key, source |
| `journey_aborted` | the journey was aborted | the step, if any, the abort reason, and its details: for an abort while data was resolved for a hook, or by or for an input adapter, the policy and hook, or the adapter, and the key; for `not a value`, the key or the event kind; for `policy could not be built`, the policy; and, when custom code caused the abort by failing, the message of its error (5.10) |

Decisions:

| Event | Decided | Also carries |
|---|---|---|
| `step_retrying` | the step goes to `MustRetry` and will be attempted again | step, the attempt that failed, the cause (`retriable failure` or `abnormal termination`), decided by |
| `step_given_up` | the step will not be attempted again and ends `Failed` | step, the last attempt, the cause (`failure`, `abnormal termination`, `retries exhausted`, or `FailWorkflow` with its reason), decided by |
| `journey_succeeded` | the journey succeeds | decided by: `default` when no step is left, or the hook that returned `FinishWorkflow` |
| `journey_failed` | the journey fails | the step that failed, its cause, its reason when there is one, and the message of its error when there is one (5.8); or the step whose hook returned `FailWorkflow` and that reason; decided by |

1. Each attempt has exactly one outcome fact: `step_succeeded`, `step_failed`, `step_skipped` or `step_abnormal_termination`.
2. Exactly one step decision MUST follow every attempt that ended in `step_failed` or `step_abnormal_termination`: `step_retrying` or `step_given_up`. A success or a skip needs no step decision.
3. `journey_aborted` is a fact: an abort is not decided, it happens, and no hook is consulted.

## 2.8 Order

Events MUST follow what happens, in this order:

1. `journey_started`.
2. For each attempt of a step:
   1. `attempt_started`;
   2. the input events, in the order the step's inputs are resolved. For an input of an adapted step: the adapter's own `optional_input_absent` events, then `input_adapter_supplied` or `input_adapter_failed`, or, when the adapter returns nothing, the events of resolving it from the data bag;
   3. the step's own `step_*` events, in the order emitted;
   4. the outcome fact.
3. After the outcome fact:
   - on Success, one `contribution_committed` per committed key, each followed by `data_overwritten` when it replaced a value;
   - on Skipped, `contributions_discarded`.
4. Then the hooks and the step decision, in the order chapter 6 defines. Each hook called contributes, in order:
   1. `optional_input_absent` for each optional request it made, for data from the workflow or from the step, that had no value;
   2. the `journey_*` events it emits, in the order emitted;
   3. its `hook_called`;
   4. its own `contribution_committed` events, each followed by `data_overwritten` when it replaced a value.

   The step decision (`step_retrying` or `step_given_up`) comes after every hook that could change it, and `step_given_up` comes before `on step failure`.
5. Finally `journey_succeeded` or `journey_failed`, after the workflow hook and its events in the same pattern as in point 4; or `journey_aborted`, which MUST be the last event of an aborted journey.

## 2.9 Events from steps and hooks

1. A step MAY emit only `step_info`, `step_warning` and `step_error`, through its step reporter. These carry the step name and attempt number.
2. A hook MAY emit only `journey_info`, `journey_warning` and `journey_error`. These carry the policy name and hook name, and, for a step hook, the step name and attempt number.
3. The author supplies only a **message** and optional **data**. The data MUST be a value (chapter 4, 4.6). The event carries its own copy, captured when it is emitted, so changing the original afterwards changes nothing. The executor MUST NOT serialize it: serializing belongs to reporters, in whatever format each chooses.
4. **Data that is not a value is illegal.** Where a language can express it, the event is not delivered, the journey is aborted with `not a value`, naming the step, or the policy and hook, and the event kind, and the emit call ends the step or hook as point 5 describes.
5. **Delivery is blocking.** An event a step or hook emits MUST be delivered to every reporter before the emit call returns: it blocks for a synchronous step or hook and is awaited for an asynchronous one. If a reporter fails during that delivery:
   1. the journey is aborted with `reporter failed`, and the abort is recorded at once;
   2. the event is not delivered to the reporters after it (2.3);
   3. the emit call ends the step or hook, by throwing an exception that belongs to the executor or, in a language without exceptions, by returning the executor's error, which the step or hook propagates. Implementations SHOULD make it impossible to catch by name;
   4. the abort stands whatever the step or hook does afterwards: nothing more it emits is delivered, and its outcome, lifecycle and contributions are ignored. There is no outcome fact, no `hook_called` and no commit, and it is never an abnormal termination;
   5. when it returns, `journey_aborted` is delivered, naming the step, whose status becomes `Aborted`.
6. Apart from point 5 and point 4, emitting an event never fails and never changes an outcome.

## 2.10 Tracing policies

Every effect of a policy is traceable in the stream: `hook_called` records each hook and the lifecycle it returned, every `journey_*` event carries the policy and hook that emitted it, every committed contribution records its source, and every decision says who decided it. There are no other policy events in tier 1.

## 2.11 Checked by

[`cases/tier-1/0011-events/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0011-events) and [`cases/tier-1/0040-events-facts-and-decisions/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0040-events-facts-and-decisions), [`0055-reporter-fails-while-custom-code-runs/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0055-reporter-fails-while-custom-code-runs), [`0063-dispatcher-factory/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0063-dispatcher-factory) and [`0064-event-data-values/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0064-event-data-values) in the conformance repository, and the event tables of every other tier 1 case.
