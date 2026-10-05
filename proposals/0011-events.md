# 0011: Events

- Issue: [#11](https://github.com/itinera-dev/spec/issues/11)
- Tier: 1
- Amended by [#40](https://github.com/itinera-dev/spec/issues/40): events record facts and decisions.
- Amended by [#49](https://github.com/itinera-dev/spec/issues/49): custom code that throws.

## Summary

How events are produced and delivered: reporters, the executor's dispatcher, the event stream, the fixed fields every event carries, the engine's catalogue and the order of its events, and what steps and hooks may emit. Builds on proposals 0002, 0008, 0009, 0010, 0024, 0027 and 0032.

## Motivation

Once decisions are moved out of business code, a flow can only be understood from what the engine reports. Operators need one stream per journey that says what happened, in order, and why, without leaking data values. Workflow authors decide where events go; tests need to capture exactly the events of a journey and nothing else; and the conformance suite judges every implementation by this stream.

## Design

### Reporters and the dispatcher

1. **A reporter** has one operation, **report**, which receives an event and decides on its own what to do with it: filter, map, forward or ignore.
2. **The workflow lists its reporters**: which reporters its journeys are reported to is the workflow's decision.
3. **The executor owns the event dispatcher.** A dispatcher has two operations: **add a reporter**, and **dispatch an event** to the reporters it holds. For each journey, the executor MUST ask the workflow instance for its reporters, add each to its dispatcher, and dispatch every event through it.
4. **A dispatcher can be given to the executor.** An executor MUST either use a dispatcher it is given when it is created, or create the default dispatcher. A dispatcher given to the executor decides what adding a reporter means; for example, a test's dispatcher may hold only the test's recording reporter and ignore the workflow's reporters.
5. **With no reporter at all**, events are produced and go nowhere.
6. **The default dispatcher** MUST call reporters in the order they were added, with every event in sequence order, and every reporter it holds MUST receive the same events. A dispatcher given to the executor defines its own delivery, except for the next rule.
7. **A reporter that throws MUST abort the journey**, with the abort reason `reporter threw`, whatever the dispatcher. The dispatcher MUST still deliver `journey_aborted` to every other reporter it holds. Reporters SHOULD be written so they cannot throw, where the language allows it.
8. **Calling the reporter is all the dispatcher does.** Reporters should be fast, but a slow one is the workflow author's choice.
9. **Reporters MAY be synchronous or asynchronous**, under the same executor capability rules as steps and hooks.

### The event stream

10. **Events are never rolled back or rewritten**, even when an attempt's contributions are discarded. An aborted journey keeps every event emitted before; **`journey_aborted` MUST be the last event** (proposal 0032).
11. **No events before a journey exists.** A workflow refused when it is built produces no event (proposal 0032).
12. **The conformance trace of a journey is its event stream.** A workflow refused before any journey exists has no stream: its trace is the refusal itself, the error listing every violation.

### Fixed fields

13. **Every event MUST carry** its **kind**, a **sequence number** increasing within the journey, a **timestamp** (UTC, ISO 8601), the **journey ID** and the **workflow name**.
14. **Events about a step** MUST also carry the **step name** and the **attempt number**.
15. **Events from or about a hook** MUST also carry the **policy name** and the **hook name**, and, for a step hook, the step and attempt that triggered it.
16. **Event kinds are snake_case.**

### The engine catalogue

17. The engine MUST emit exactly these events:

    | Event | When | Also carries |
    |---|---|---|
    | `journey_started` | the journey starts | the initial keys, never their values |
    | `attempt_started` | an attempt of a step starts | step, attempt |
    | `input_adapter_supplied` | an input adapter supplies a value | step, key |
    | `input_adapter_failed` | an input adapter fails | step, key |
    | `optional_input_absent` | an optional input has no value | step, key |
    | `step_succeeded` | a step succeeds | step, attempt |
    | `step_failed` | a step fails | step, attempt, retriable, reason (code, message, details) |
    | `step_skipped` | a step is skipped | step, attempt, reason, if any |
    | `abnormal_termination` | an error escapes a running step | step, attempt, error message |
    | `hook_called` | a hook returns | policy, hook, step and attempt if any, the lifecycle returned or none |
    | `contribution_committed` | a contribution reaches the data bag | key, source: the step, or the policy and hook |
    | `contributions_discarded` | a skipped step's contributions are discarded | step, attempt |
    | `data_overwritten` | a committed key replaces an earlier value | key, source |
    | `journey_succeeded` | the journey succeeds | |
    | `journey_failed` | the journey fails | the step and its reason or cause, or the step whose hook returned `FailWorkflow` and its reason |
    | `journey_aborted` | the journey is aborted | the step, if any, the abort reason and its details |

18. **Order.** Events MUST follow what happens, in this order:
    1. `journey_started`;
    2. for each attempt of a step: `attempt_started`; the input events in the order the inputs are resolved; the step's own `step_*` events, in the order emitted; then the outcome event (`step_succeeded`, `step_failed`, `step_skipped` or `abnormal_termination`);
    3. on success, one `contribution_committed` per committed key, each followed by `data_overwritten` when it replaced a value; for a skip, `contributions_discarded`;
    4. then, for each hook called, the `journey_*` events it emits, its `hook_called`, then its own `contribution_committed` and `data_overwritten` events;
    5. finally `journey_succeeded`, `journey_failed` or `journey_aborted`, after any workflow hooks and their events, in the same pattern.
19. **No separate status-change events.** Every step status follows from these events.
20. **Engine events MUST NOT carry data values**, only keys. Reasons and error messages are carried because their authors wrote them.

### What steps and hooks emit

21. **Steps** MAY emit only `step_info`, `step_warning` and `step_error`, through their step reporter. **Hooks** MAY emit only `journey_info`, `journey_warning` and `journey_error`.
22. **Authors supply only a message and optional data.** The data MUST be serializable and is serialized when the event is emitted; if it cannot be serialized, the event MUST still be delivered, with a marker saying so.
23. **Emitting never fails and never changes an outcome.**

### Policy events

24. **Every policy's effect is traceable** through `hook_called`, the policy and hook carried by every `journey_*` event a hook emits, and the source of every contribution. No further policy events are needed in tier 1.

## Events

This proposal defines the catalogue above.

## Conformance

The conformance cases for this proposal must check that:

1. with the default dispatcher, every reporter the workflow lists receives the same events;
2. a dispatcher given to the executor is used, and the workflow's reporters are added to it;
3. a given dispatcher that ignores additions delivers only to its own reporters;
4. a reporter that throws aborts the journey, and the other reporters still receive `journey_aborted`;
5. every event carries the journey ID and the workflow name, with increasing sequence numbers;
6. a complete journey produces exactly the catalogue's events, in the defined order;
7. a failed attempt's events remain, and `step_failed` carries the retriable flag and the reason;
8. `hook_called` records the lifecycle returned, and `journey_failed` the step and the reason;
9. contributions record their source, and `abnormal_termination` is its own event;
10. engine events never carry data values;
11. step and hook events carry their message, data, step, attempt, policy and hook;
12. data that cannot be serialized still produces the event, with the marker.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0011-events/`. The event names used provisionally by earlier cases are replaced by the names in item 17.

## Alternatives considered

- **Executors configuring reporters.** Rejected: which reporters matter is the workflow's decision. The executor owns only the delivery.
- **Reporter failures isolated from the flow.** Rejected: a reporter that throws aborts the journey, so a broken reporter is never silently ignored.
- **Status-change events.** Rejected: every status follows from the outcome events, and extra events add noise.
- **Carrying data values in engine events.** Rejected: values may be large or sensitive; keys say what changed.
- **An event when a workflow is refused before any journey.** Rejected: there is no journey to report on.

## Open questions

- Events for later tiers: switches, transfers, sub workflows, pending steps and resumed journeys.
