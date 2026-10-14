# 2. Events

Specification 0.1. Written from proposal [0011](../proposals/0011-events.md) as amended by [0040](../proposals/0040-events-facts-and-decisions.md) and [0041](../proposals/0041-hook-data-from-the-data-bag.md), [0054](../proposals/0054-how-custom-code-fails.md), [0055](../proposals/0055-reporter-fails-while-custom-code-runs.md), [0063](../proposals/0063-dispatcher-factory.md), [0064](../proposals/0064-event-data-values.md), [0081](../proposals/0081-reporter-failure-stops-delivery.md), [0083](../proposals/0083-what-failures-carry.md), [0095](../proposals/0095-services-provided-to-steps.md), [0102](../proposals/0102-global-metadata.md), [0104](../proposals/0104-service-implementation-on-events.md) and [0106](../proposals/0106-retry-budget-alone.md), with the event-stream rules of [0032](../proposals/0032-configuration-errors.md), the rules of [0049](../proposals/0049-custom-code-that-throws.md) for reporters that fail, and the events introduced by [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md), [0009](../proposals/0009-running-a-workflow.md) and [0003](../proposals/0003-rescue-steps.md).

Events come first because every conformance case observes a journey through its event stream. The events about steps, hooks and data are listed here in full; what makes each happen is defined in the chapters that follow.

## 2.1 Reporters

1. A **reporter** has two operations: **report**, which receives one event, and **receive global metadata**, which receives one entry of the journey's global metadata (2.11). It decides on its own what to do with each: filter it, map it, forward it, keep it or ignore it.
2. A workflow lists its reporters. Which reporters a journey is reported to is the workflow's decision, never the executor's.
3. A reporter MAY be synchronous or asynchronous, under the executor's execution modes (chapter 5).
4. **Reporters handle their own trouble.** A reporter SHOULD NOT fail. A reporter that cannot report an event SHOULD handle that itself, as it chooses (logging it, dropping the event, keeping it to retry later), and return normally, so that the reporters after it still receive the event. A failure that escapes a reporter aborts the journey (2.3).

## 2.2 The dispatcher

1. An **event dispatcher** has three operations: **add a reporter**, **dispatch an event** to the reporters it holds, and **set global metadata**, which passes one entry to the reporters it holds (2.11).
2. **A dispatcher factory** creates dispatchers. An executor is given a dispatcher factory when it is created, or uses the default factory, which creates the default dispatcher.
3. **One dispatcher per journey.** For each journey, the executor MUST call its factory once, before the journey starts, add the workflow instance's reporters to the dispatcher it returns, and dispatch every event of the journey through it. When the journey ends, the executor drops that dispatcher, with every reporter added to it, so a reporter never receives another journey's events.
4. **The default dispatcher** MUST call reporters in the order they were added, with every event in sequence order and every entry of global metadata in the order it was set, and every reporter it holds MUST receive the same events and the same entries, except as 2.3 defines when a reporter fails.
5. **Other dispatchers** decide what adding a reporter means and how they deliver events and global metadata; for example, a test's factory may return dispatchers that all hold the test's recording reporter and ignore the workflow's reporters.
6. **With no reporter at all**, events are still produced, and go nowhere.
7. **Calling reporters is all a dispatcher does.** A slow reporter slows the journey; that is the workflow author's choice.

## 2.3 A reporter or dispatcher that fails

Custom code fails by throwing or by returning an error (chapter 5, 5.10). A reporter or a dispatcher cannot report an error and carry on, so its failure aborts the journey. What this section says of an event also holds for an entry of global metadata set while the journey runs (2.11); one set before the journey starts is covered by point 5.

1. A reporter that fails on any event except `journey_aborted` MUST abort the journey with the abort reason `reporter failed`, whatever the dispatcher.
2. The event it failed on MUST NOT be delivered to the reporters after it. Then `journey_aborted` MUST be delivered to every reporter the dispatcher holds except the one that failed. This holds whatever the dispatcher, for the reporters added to it for the journey: once a reporter has failed, none of them receives any further event except `journey_aborted`, and the one that failed receives nothing more, even if the dispatcher would go on delivering.
3. A dispatcher that fails while dispatching an event other than `journey_aborted` aborts the journey in the same way, with `reporter failed`.
4. A reporter or a dispatcher that fails while `journey_aborted` is being delivered MUST be ignored: the journey is already aborted, its first abort reason stands, no further event is emitted, and delivery continues to the remaining reporters.
5. A dispatcher factory that fails, or a dispatcher that fails while the journey's reporters are added, or a reporter or dispatcher that fails while the global metadata set before the journey starts is passed on (2.11 point 6), is a refusal, before any event (chapter 5, 5.10).
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
3. An event from or about a hook MUST also carry the **policy name** and the **hook name**, and, for a step hook, the step name and attempt number that triggered it. An event from or about a service provider MUST also carry the **provider name**, with the step name and attempt number of the step being built.
4. `journey_failed` and `journey_aborted` are events about the journey, not about a step. They explain why the journey ended: `journey_failed` names the step that failed, and `journey_aborted` the step during which it was aborted, if any, but neither carries an attempt number.

## 2.6 Facts and decisions

1. Engine events are of two kinds.
   - **A fact** records something that happened, when it happens. It MUST NOT depend on a hook.
   - **A decision** records what the executor decided to do next. It MUST be emitted only once the decision is final, after every hook that could change it has returned, and MUST carry **decided by**: either `default`, the executor's own rule, or the policy and hook whose returned lifecycle decided it.
2. So the stream always reads: what happened, the hooks that were consulted, what was decided.
3. There are no separate events for step status changes: every step status follows from the facts and decisions below.
4. Engine events MUST NOT carry data values, only keys. Reasons and error messages are carried, because their authors wrote them, and so is the data a service provider supplies with an instance for `service_provided` (chapter 4, 4.9).

## 2.7 The engine catalogue

The executor MUST emit exactly these engine events, and no others, in tier 1.

Facts:

| Event | When | Also carries |
|---|---|---|
| `journey_started` | the journey starts | the initial keys of the data bag, never their values |
| `attempt_started` | an attempt of a step starts | step, attempt |
| `input_adapter_supplied` | an input adapter supplies a value for a step's input | step, key, adapter |
| `input_adapter_failed` | an input adapter fails, or supplies something that is not a value, for a step's input | step, key, adapter |
| `optional_input_absent` | an optional request has no value: a step's input, a hook's data from the workflow or from the step, a rescue step's data from the step it rescues (chapter 7), or an input adapter's or a service provider's data from the workflow | step, key; the policy and hook, the adapter, or the provider, when the request is theirs |
| `service_provided` | a service provider returned an instance for a step's service | step, attempt, key, provider, the implementation it made, and the data it supplied with the instance, if any (chapter 4, 4.9) |
| `service_provider_failed` | a service provider failed, or returned nothing, for a step's service | step, attempt, key, provider |
| `global_metadata_set` | a step, hook or service provider set an entry of global metadata, after the reporters received it (2.11) | the key, never its value; the step and attempt, when there is one; and the policy and hook, or the provider, when they set it |
| `step_succeeded` | the attempt reported Success | step, attempt |
| `step_failed` | the attempt reported Failure | step, attempt, reason (code, message, details) |
| `step_skipped` | the attempt reported Skipped | step, attempt, the reason, if any |
| `step_abnormal_termination` | an error escaped the running step | step, attempt, the error's message |
| `hook_called` | a hook returned without throwing | policy, hook, step and attempt if any, the lifecycle returned or none |
| `contribution_committed` | a contribution reached the data bag | key, source: the step, or the policy and hook |
| `contributions_discarded` | a skipped step's contributions were discarded | step, attempt |
| `data_overwritten` | a committed key replaced an earlier value | key, source |
| `journey_aborted` | the journey was aborted | the step, if any, the abort reason, and its details: for an abort while data was resolved for a hook, or by or for an input adapter, or for a service provider, the policy and hook, the adapter, or the provider, and the key, or, for a hook's or a rescue step's request for the failure reason or the error, which of the two was requested; for `not a value`, the policy and hook or the provider when the value was theirs, and the key or the event kind; for `reserved metadata key`, the policy and hook or the provider when they set it, and the key; for `policy could not be built`, the policy; for `step could not be built` because a service provider failed, the provider; and, when custom code caused the abort by failing, the message of its error (5.10) |

Decisions:

| Event | Decided | Also carries |
|---|---|---|
| `step_retrying` | the step goes to `MustRetry` and will be attempted again | step, the attempt that failed, the cause (`failure` or `abnormal termination`), decided by |
| `step_rescuing` | the step goes to `Rescuing`: its rescue step is inserted before it (chapter 7) | step, the attempt that failed, the cause (`failure` or `abnormal termination`), the rescue step's name, decided by |
| `step_rescued` | the rescue step succeeded, and the step will be attempted again (chapter 7) | step, the attempt that failed, the rescue step's name, decided by |
| `step_given_up` | the step will not be attempted again and ends `Failed` | step, the last attempt, the cause (`failure`, `abnormal termination`, `retries exhausted`, or `FailWorkflow` with its reason), decided by |
| `journey_succeeded` | the journey succeeds | decided by: `default` when no step is left, or the hook that returned `FinishWorkflow` |
| `journey_failed` | the journey fails | the step that failed, its cause, its reason when there is one, and the message of its error when there is one (5.8); or the step whose hook returned `FailWorkflow` and that reason; decided by |

1. Each attempt has exactly one outcome fact: `step_succeeded`, `step_failed`, `step_skipped` or `step_abnormal_termination`.
2. Exactly one step decision MUST follow every attempt that ended in `step_failed` or `step_abnormal_termination`: `step_retrying`, `step_rescuing` or `step_given_up`. A success or a skip needs no step decision. `step_rescued` follows the success of a rescue step (chapter 7).
3. `journey_aborted` is a fact: an abort is not decided, it happens, and no hook is consulted.

## 2.8 Order

Events MUST follow what happens, in this order:

1. `journey_started`.
2. For each attempt of a step:
   1. `attempt_started`;
   2. the input events, in the order the step's inputs are resolved. For an input of an adapted step: the adapter's own `optional_input_absent` events, then `input_adapter_supplied` or `input_adapter_failed`, or, when the adapter returns nothing, the events of resolving it from the data bag;
   3. the service events, in the order the step's services are resolved (chapter 4, 4.9). For each service whose provider is called: the provider's own `optional_input_absent` events, then the `journey_*` events it emits and its `global_metadata_set` events, in the order they happen, then `service_provided` or `service_provider_failed`. A service of journey scope made for an earlier attempt has no event;
   4. the step's own `step_*` events and `global_metadata_set` events, in the order they happen;
   5. the outcome fact.
3. After the outcome fact:
   - on Success, one `contribution_committed` per committed key, each followed by `data_overwritten` when it replaced a value;
   - on Skipped, `contributions_discarded`.
4. Then the hooks and the step decision, in the order chapter 6 defines. Each hook called contributes, in order:
   1. `optional_input_absent` for each optional request it made, for data from the workflow or from the step, that had no value, in the order its requests are resolved (chapter 6, 6.6 point 4);
   2. the `journey_*` events it emits and its `global_metadata_set` events, in the order they happen;
   3. its `hook_called`;
   4. its own `contribution_committed` events, each followed by `data_overwritten` when it replaced a value.

   The step decision (`step_retrying`, `step_rescuing` or `step_given_up`) comes after every hook that could change it, and `step_given_up` comes before `on step failure`. After `step_rescuing`, the rescue step's attempt follows, in the same pattern as any attempt, then `step_rescued` once it succeeded (chapter 7, 7.6).
5. Finally `journey_succeeded` or `journey_failed`, after the workflow hook and its events in the same pattern as in point 4; or `journey_aborted`, which MUST be the last event of an aborted journey.

## 2.9 Events from steps and hooks

1. A step MAY emit only `step_info`, `step_warning` and `step_error`, through its step reporter. These carry the step name and attempt number.
2. A hook or a service provider MAY emit only `journey_info`, `journey_warning` and `journey_error`. These carry the policy name and hook name, and, for a step hook, the step name and attempt number; or, from a service provider, the provider name, with the step name and attempt number of the step being built. What this section says of a hook also holds for a service provider.
3. The author supplies only a **message** and optional **data**. The data MUST be a value (chapter 4, 4.6). The event carries its own copy, captured when it is emitted, so changing the original afterwards changes nothing. The executor MUST NOT serialize it: serializing belongs to reporters, in whatever format each chooses.
4. **Data that is not a value is illegal.** Where a language can express it, the event is not delivered, the journey is aborted with `not a value`, naming the step, or the policy and hook, or the service provider and the step being built, and the event kind, and the emit call ends the step or hook as point 5 describes.
5. **Delivery is blocking.** An event a step or hook emits MUST be delivered to every reporter before the emit call returns: it blocks for a synchronous step or hook and is awaited for an asynchronous one. If a reporter fails during that delivery:
   1. the journey is aborted with `reporter failed`, and the abort is recorded at once;
   2. the event is not delivered to the reporters after it (2.3);
   3. the emit call ends the step or hook, by throwing an exception that belongs to the executor or, in a language without exceptions, by returning the executor's error, which the step or hook propagates. Implementations SHOULD make it impossible to catch by name;
   4. the abort stands whatever the step or hook does afterwards: nothing more it emits is delivered, and its outcome, lifecycle and contributions are ignored. There is no outcome fact, no `hook_called` and no commit, and it is never an abnormal termination;
   5. when it returns, `journey_aborted` is delivered, naming the step, whose status becomes `Aborted` if it was `Executing` or `MustRetry` (chapter 5, 5.7); when the step is a rescue step, the step it rescues becomes `Aborted` too (chapter 7, 7.5).
6. Apart from point 5 and point 4, emitting an event never fails and never changes an outcome.
7. Setting global metadata through a step reporter, or through a hook's or service provider's means of emitting events, follows the same rules (2.11).

## 2.10 Tracing policies

Every effect of a policy is traceable in the stream: `hook_called` records each hook and the lifecycle it returned, every `journey_*` event a hook emits carries its policy and hook, every committed contribution records its source, and every decision says who decided it. A `journey_*` event a service provider emits carries the provider's name instead (2.9 point 2). There are no other policy events in tier 1.

## 2.11 Global metadata

1. **What it is.** The **global metadata** of a journey is a set of entries, each a **key**, a string, and a **value** (chapter 4, 4.6). It belongs to the journey and exists for its reporters. It never enters the data bag or the result, and no step, hook, input adapter or service provider can read it.
2. **One entry per key.** Setting a key that already has an entry replaces its value. An entry is never removed.
3. **The journey ID.** The key `journey_id` holds the journey ID, as a string. Only the executor sets it.
4. **What reporters do with it.** A reporter MUST accept every entry it receives, and decides on its own what to do with it. A reporter SHOULD keep the latest value of each key until the journey ends, so that it can use them with the events that follow.
5. **In order with the events.** Entries and events reach reporters in the order they happen: an entry set while the journey runs reaches them after every event emitted before it was set, and before every event emitted after it.
6. **Before the journey starts.** After adding the reporters to the dispatcher (2.2 point 3) and before `journey_started`, the executor MUST set `journey_id` to the journey ID, then each entry of the workflow instance's initial global metadata (chapter 5, 5.1), in the order its keys were first set. Neither emits an event. A reporter or the dispatcher that fails while they are set is a refusal, with no journey and no event (2.3 point 5).
7. **While the journey runs.** A step, through its step reporter, and a hook and a service provider, through their means of emitting events, MAY set global metadata, one entry at a time. A rescue step is a step. Input adapters, role operations and constructors cannot.
   1. The value is captured when it is set: changing the original afterwards changes nothing.
   2. The entry is set on the dispatcher, and so passed to the reporters, and then `global_metadata_set` is emitted, all before the call returns. The call blocks for a synchronous step, hook or provider and is awaited for an asynchronous one.
   3. An entry stands from the moment it is set, whatever the code that set it does afterwards. A failed, retried or skipped attempt, an abnormal termination and an abort take back nothing, as events are never rolled back (2.4).
   4. A value that is not a value is illegal, as in 2.9 point 4: the journey is aborted with `not a value`, naming the step, or the policy and hook, or the service provider and the step being built, and the key. The entry reaches no reporter, `global_metadata_set` is not emitted, and the call ends the step, hook or provider as 2.9 point 5 describes.
   5. Setting `journey_id` is illegal in the same way, and aborts the journey with `reserved metadata key`, naming the same.
   6. A reporter or the dispatcher that fails on the entry aborts the journey with `reporter failed`, as on an event emitted by the same code (2.3 and 2.9 point 5): the entry reaches no reporter after the one that failed, `global_metadata_set` is not emitted, and the call ends the step, hook or provider.
   7. A step reporter whose attempt has ended, or a hook's or provider's means of emitting events after its call has returned, sets nothing, as chapter 4, 4.6 point 7 says of a closed handle.
8. **When the journey ends,** the global metadata goes with the dispatcher and its reporters (2.2 point 3).

## 2.12 Checked by

[`cases/tier-1/0011-events/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0011-events) and [`cases/tier-1/0040-events-facts-and-decisions/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0040-events-facts-and-decisions), [`0055-reporter-fails-while-custom-code-runs/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0055-reporter-fails-while-custom-code-runs), [`0063-dispatcher-factory/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0063-dispatcher-factory) and [`0064-event-data-values/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0064-event-data-values) and [`0102-global-metadata/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0102-global-metadata) in the conformance repository, and the event tables of every other tier 1 case.
