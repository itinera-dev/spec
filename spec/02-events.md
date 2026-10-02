# 2. Events

Specification 0.1. Written from proposal [0011](../proposals/0011-events.md), with the event-stream rules of [0032](../proposals/0032-configuration-errors.md) and the events introduced by [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md) and [0009](../proposals/0009-running-a-workflow.md).

Events come first because every conformance case observes a journey through its event stream. The events about steps, hooks and data are listed here in full; what makes each happen is defined in the chapters that follow.

## 2.1 Reporters

1. A **reporter** has one operation, **report**, which receives one event. It decides on its own what to do with it: filter it, map it, forward it or ignore it.
2. A workflow lists its reporters. Which reporters a journey is reported to is the workflow's decision, never the executor's.
3. A reporter MAY be synchronous or asynchronous, under the executor's execution modes (chapter 5).
4. Reporters SHOULD be written so that they cannot throw, where the language allows it.

## 2.2 The dispatcher

1. The executor owns an **event dispatcher**, with two operations: **add a reporter**, and **dispatch an event** to the reporters it holds.
2. An executor MUST either use a dispatcher it is given when it is created, or create the default dispatcher.
3. For each journey, the executor MUST ask the workflow instance for its reporters, add each to the dispatcher, and dispatch every event of the journey through it.
4. **The default dispatcher** MUST be created afresh for each journey. It MUST call reporters in the order they were added, with every event in sequence order, and every reporter it holds MUST receive the same events.
5. **A dispatcher given to the executor** decides what adding a reporter means and how it delivers; for example, a test's dispatcher may hold only the test's recording reporter and ignore the workflow's reporters. It is used for each journey in turn, and the workflow's reporters added for a journey MUST be removed when that journey ends, so that they never receive another journey's events.
6. **With no reporter at all**, events are still produced, and go nowhere.
7. **Calling reporters is all a dispatcher does.** A slow reporter slows the journey; that is the workflow author's choice.

## 2.3 A reporter that throws

1. A reporter that throws or panics MUST abort the journey with the abort reason `reporter threw`, whatever the dispatcher.
2. The dispatcher MUST still deliver `journey_aborted` to every other reporter it holds.

## 2.4 The event stream

1. The **event stream** of a journey is every event dispatched for it, in order.
2. Events MUST NOT be rolled back or rewritten, even when an attempt's contributions are discarded.
3. An aborted journey keeps every event emitted before the abort, and `journey_aborted` MUST be its last event.
4. A workflow refused when it is built, before any journey exists, MUST NOT produce any event (chapter 3).
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

## 2.6 The engine catalogue

The executor MUST emit exactly these engine events, and no others, in tier 1.

| Event | When | Also carries |
|---|---|---|
| `journey_started` | the journey starts | the initial keys of the data bag, never their values |
| `attempt_started` | an attempt of a step starts | step, attempt |
| `input_adapter_supplied` | an input adapter supplies a value | step, key |
| `input_adapter_failed` | an input adapter fails | step, key, and the policy and hook when resolving a hook's request |
| `optional_input_absent` | an optional request has no value | step, key |
| `step_succeeded` | an attempt ends in Success | step, attempt |
| `step_failed` | an attempt ends in Failure | step, attempt, retriable, reason (code, message, details) |
| `step_skipped` | an attempt ends in Skipped | step, attempt, the reason, if any |
| `abnormal_termination` | an error escapes a running step | step, attempt, the error's message |
| `hook_called` | a hook returns without throwing | policy, hook, step and attempt if any, the lifecycle returned or none |
| `contribution_committed` | a contribution reaches the data bag | key, source: the step, or the policy and hook |
| `contributions_discarded` | a skipped step's contributions are discarded | step, attempt |
| `data_overwritten` | a committed key replaces an earlier value | key, source |
| `journey_succeeded` | the journey succeeds | |
| `journey_failed` | the journey fails | the step and its reason or cause; or the step whose hook returned `FailWorkflow` and its reason |
| `journey_aborted` | the journey is aborted | the step, if any, the abort reason, and its details: every violation for `invalid configuration`; for `hook threw`, the policy and hook, and the key being resolved if the hook failed while its data was resolved |

1. Each attempt has exactly one outcome event: `step_succeeded`, `step_failed`, `step_skipped` or `abnormal_termination`. An abnormal termination is never also reported as `step_failed`.
2. There are no separate events for step status changes: every step status follows from the events above.
3. Engine events MUST NOT carry data values, only keys. Reasons and error messages are carried, because their authors wrote them.

## 2.7 Order

Events MUST follow what happens, in this order:

1. `journey_started`.
2. For each attempt of a step:
   1. `attempt_started`;
   2. the input events (`input_adapter_supplied`, `input_adapter_failed`, `optional_input_absent`), in the order the step's inputs are resolved;
   3. the step's own `step_*` events, in the order emitted;
   4. the outcome event.
3. After the outcome event:
   - on Success, one `contribution_committed` per committed key, each followed by `data_overwritten` when it replaced a value;
   - on Skipped, `contributions_discarded`.
4. Then, for each hook called, in the order the hooks are called (chapter 6):
   1. the input events of the data it requests from the workflow, in the order resolved;
   2. the `journey_*` events it emits, in the order emitted;
   3. its `hook_called`;
   4. its own `contribution_committed` events, each followed by `data_overwritten` when it replaced a value.
5. Finally `journey_succeeded`, `journey_failed` or `journey_aborted`. For a journey that succeeds or fails, the workflow hook's events come first, in the same pattern as in point 4.

## 2.8 Events from steps and hooks

1. A step MAY emit only `step_info`, `step_warning` and `step_error`, through its step reporter. These carry the step name and attempt number.
2. A hook MAY emit only `journey_info`, `journey_warning` and `journey_error`. These carry the policy name and hook name, and, for a step hook, the step name and attempt number.
3. The author supplies only a **message** and optional **data**. The data MUST be serializable and is serialized when the event is emitted; if it cannot be serialized, the event MUST still be delivered, with a marker saying so.
4. Emitting an event never fails and never changes an outcome.

## 2.9 Tracing policies

Every effect of a policy is traceable in the stream: `hook_called` records each hook and the lifecycle it returned, every `journey_*` event carries the policy and hook that emitted it, and every committed contribution records its source. There are no other policy events in tier 1.

## 2.10 Checked by

[`cases/tier-1/0011-events/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0011-events) in the conformance repository, and the event tables of every other tier 1 case.
