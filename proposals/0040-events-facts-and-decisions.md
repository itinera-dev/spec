# 0040: Events record facts and decisions

- Issue: [#40](https://github.com/itinera-dev/spec/issues/40)
- Tier: 1
- Amends: [0011: Events](0011-events.md)

## Summary

Engine events are of two kinds: **facts**, which record what happened, and **decisions**, which record what the executor decided to do next and who decided it. A decision event is emitted only once the decision is final, after every hook that could change it. After every attempt that failed or terminated abnormally, one decision says whether the step is retried (`step_retrying`) or given up (`step_given_up`). `abnormal_termination` is renamed `step_abnormal_termination`.

## Motivation

In proposal 0011, the stream records what each attempt reported, but not what was decided about the step afterwards: whether it was retried, or given up because the failure was not retriable, the abnormal termination was not retriable, the retry budget ran out, or a hook decided so. Some events record things that happened and others record decisions, and decisions depend on hooks, so it was unclear whether an event comes before or after the hooks. One event for each thing that happened, and one for each decision taken, makes the stream read in the order things occur: what happened, the hooks consulted, what was decided.

## Design

### Facts and decisions

1. **A fact event** records something that happened, when it happens. It MUST NOT depend on a hook.
2. **A decision event** records what the executor decided to do next. It MUST be emitted only once the decision is final: after every hook that could change it has returned. It MUST carry **decided by**: either `default`, the executor's own rule, or the policy and hook whose returned lifecycle decided it.

### The catalogue

Facts:

| Event | When | Also carries |
|---|---|---|
| `journey_started` | the journey starts | the initial keys, never their values |
| `attempt_started` | an attempt of a step starts | step, attempt |
| `input_adapter_supplied` | an input adapter supplies a value | step, key |
| `input_adapter_failed` | an input adapter fails | step, key |
| `optional_input_absent` | an optional request has no value | step, key |
| `step_succeeded` | the attempt reported Success | step, attempt |
| `step_failed` | the attempt reported Failure, as in proposal 0011 | step, attempt, retriable, reason |
| `step_skipped` | the attempt reported Skipped | step, attempt, reason, if any |
| `step_abnormal_termination` | an error escaped the running step (renames `abnormal_termination`) | step, attempt, the error's message |
| `hook_called` | a hook returned | policy, hook, step and attempt if any, the lifecycle returned or none |
| `contribution_committed` | a contribution reached the data bag | key, source |
| `contributions_discarded` | a skipped step's contributions were discarded | step, attempt |
| `data_overwritten` | a committed key replaced an earlier value | key, source |
| `journey_aborted` | the journey was aborted | the step, if any, the abort reason and its details |

Decisions:

| Event | Decided | Also carries |
|---|---|---|
| `step_retrying` | the step goes to `MustRetry` and will be attempted again | step, the attempt that failed, the cause (`retriable failure` or `abnormal termination`), decided by |
| `step_given_up` | the step will not be attempted again and ends `Failed` | step, the last attempt, the cause (`failure`, `abnormal termination`, `retries exhausted`, or `FailWorkflow` with its reason), decided by |
| `journey_succeeded` | the journey succeeds | decided by: `default` when no step is left, or the hook that returned `FinishWorkflow` |
| `journey_failed` | the journey fails | the step and its cause and reason, or the `FailWorkflow` reason; decided by |

`journey_aborted` is a fact: an abort is not decided, it happens, and no hook is consulted.

### When decisions are emitted

3. **Exactly one decision MUST follow every attempt that ended in `step_failed` or `step_abnormal_termination`:** `step_retrying` or `step_given_up`. A success or a skip needs no step decision.
4. **`step_retrying`** follows the hooks of the attempt: after `on step abnormal termination`, if any, and after `on step retry` returns nothing.
5. **`step_given_up`** follows the hooks that could still change the step's fate (`on step abnormal termination`, or `on step retry` returning `FailWorkflow`), and MUST come **before** `on step failure`: the failure hook is called because the step was given up, and decides only the journey's fate.
6. **A retry hook or abnormal termination hook returning `FailWorkflow`** emits `step_given_up` with the cause `FailWorkflow`, decided by that policy and hook, then `journey_failed`. `on step failure` is not called (proposal 0010).
7. **`FailWorkflow` from `on step success`** emits no `step_given_up`: the step stays `Succeeded`.
8. **`journey_succeeded` and `journey_failed`** come last, after the workflow hooks, as in proposal 0011.

### Examples

1. **A retriable failure, then success:** `attempt_started` (attempt 1), `step_failed` (retriable), `hook_called` (`on step retry`), `step_retrying` (decided by `default`), `attempt_started` (attempt 2), `step_succeeded`, the commits, `journey_succeeded`.
2. **An abnormal termination that is not retriable:** `attempt_started` (attempt 1), `step_abnormal_termination`, `hook_called` (`on step abnormal termination`), `step_given_up` (cause `abnormal termination`, decided by `default`), `hook_called` (`on step failure`), `journey_failed` (decided by `default`).
3. **Retries exhausted:** the last `step_failed` (retriable), `step_given_up` (cause `retries exhausted`), `hook_called` (`on step failure`), `journey_failed`.

### What this replaces

In proposal 0011: `abnormal_termination` in the catalogue (item 17) becomes `step_abnormal_termination`, the decision events and the "decided by" field are added, and the order (item 18) gains the decision events above. `step_failed` keeps its name and meaning. Item 19, no separate status-change events, still holds: `step_retrying` and `step_given_up` record decisions, and are the only step decisions.

## Events

Adds `step_retrying`, `step_given_up` and the "decided by" field of decision events; renames `abnormal_termination` to `step_abnormal_termination`.

## Conformance

The conformance cases for this proposal must check that:

1. a retried failure emits `step_failed`, then `step_retrying` after the retry hook;
2. a failure that is not retriable emits `step_failed`, then `step_given_up` with the cause `failure`;
3. retries exhausted emit `step_given_up` with the cause `retries exhausted`, before the failure hook's `hook_called`;
4. an abnormal termination that is not retriable emits `step_abnormal_termination`, then `step_given_up` with the cause `abnormal termination`;
5. `FailWorkflow` from a retry hook emits `step_given_up` and `journey_failed`, both decided by that hook;
6. `journey_succeeded` and `journey_failed` say who decided: the hook that returned `FinishWorkflow` or `FailWorkflow`, otherwise `default`.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0040-events-facts-and-decisions/`.

## Alternatives considered

- **One outcome event per attempt and nothing more** (proposal 0011 as accepted). Rejected: what was decided after the attempt had to be inferred from the retry rules.
- **Renaming the attempt's failure `step_attempt_failed` and using `step_failed` for the decision.** Rejected: "the step failed" reads as what the step reported, which is a fact; the decision gets its own name.
- **`step_not_retried`, `step_abandoned` or `step_ended_failed` for the decision.** Rejected in favour of `step_given_up`, the word proposal 0010 already uses.

## Open questions

None.
