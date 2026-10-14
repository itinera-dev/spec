# 0106: The retry budget alone decides retries

- Issue: [#106](https://github.com/itinera-dev/spec/issues/106)
- Tier: 1
- Amends: [0003](0003-rescue-steps.md), [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0024](0024-abnormal-termination-retriable.md), [0040](0040-events-facts-and-decisions.md) and [0083](0083-what-failures-carry.md)

## Summary

A failure no longer carries a **retriable** flag. Whether a step is tried again is decided by its **retry budget**, then by its hooks: a failure with budget left calls `on step retry` and the step is attempted again, unless a hook returns `FailWorkflow`; once the budget is spent the step is given up. A budget of 0 means the step is never retried.

An abnormal termination follows the same rule. The step descriptor's setting `abnormal termination retriable`, false by default, becomes **`retry on abnormal termination`**, **true** by default: a step turns it off when retrying an error it did not anticipate is unsafe.

Amends proposal 0008 (the retriable flag, the causes hooks receive), proposal 0024 (the setting and its default), proposal 0009 (what is retried), proposal 0010 (which hooks run), proposal 0011 (`step_failed`), proposal 0040 (the causes of the step decisions), proposal 0083 (what `retries exhausted` carries) and proposal 0003 (the causes of a rescue). In the chapters it changes 1.2, 1.4, 2.7, 3.2, 4.3, 4.4, 4.6, 4.8, 5.6, 5.8, 6.2 and chapter 7.

## Motivation

Today three things decide whether a step is tried again: the step's flag on each failure, the step descriptor's retry budget, and, for abnormal terminations, a second setting. The flag asks a step to judge whether trying again might help, which is a flow decision dressed as a fact, and it breaks the principle that steps know nothing about the flow. A step that forgets the flag is never retried whatever its budget says, and a reader of the workflow cannot tell from the declaration alone whether a step with a budget of 3 will ever be retried.

The workflow already states each step's budget, and the hooks already see the cause, the reason and the error: `on step retry` may return `FailWorkflow` when the reason says trying again is pointless. One rule is enough: a step with budget left is retried, and its policies decide when that should stop early.

The same reading applies to abnormal terminations, with one difference kept from proposal 0024: an error the step did not anticipate may already have had side effects, so the descriptor keeps a switch to turn their retries off. With failures retried by budget alone, the switch is on by default, so a budget means the same for both.

## Design

### Outcomes

1. **A failure carries a reason, and nothing else.** A step reports Success, Failure or Skipped. A failure carries its reason (a code, an optional message and optional details); it carries no retriable flag, and nothing a step reports asks for a retry.

### Retries

1. **The retry budget decides.** After an attempt that reported a failure, or ended in an abnormal termination while the descriptor's `retry on abnormal termination` is on, the step MUST go to `MustRetry` while its retry budget allows another attempt, unless a hook returns `FailWorkflow`. Failures and abnormal terminations use the same budget.
2. **A budget of 0 means no retries.** The budget is still per step, explicit and 0 by default.
3. **When the budget is spent**, after one or more retries, the step is given up with the cause `retries exhausted`. The retry hook MUST NOT be called when no retry is possible.
4. **When the step had no budget**, it is given up at once with the cause of its only attempt: `failure` or `abnormal termination`. `retries exhausted` is reported only when a budget greater than 0 was spent.
5. **`retry on abnormal termination`** is a setting of each step descriptor, **true** unless the descriptor says otherwise, with no workflow-wide default. When it is off, an abnormal termination makes the step `Failed` at once with the cause `abnormal termination`, whatever its budget, as proposal 0024 did by default.
6. **Unchanged:** an abnormal termination never aborts the journey; a retry happens immediately; a step with a rescue step has a retry budget of 0 and is rescued after its first failed attempt, whatever its outcome or its `retry on abnormal termination` setting.

### Hooks

After each attempt's outcome, the executor calls the hooks and records the step decision in this order:

1. **Success:** `on step success`.
2. **A failure, with budget left:** `on step retry`, with the cause `failure`; then `step_retrying` and the next attempt.
3. **A failure, without budget left:** `step_given_up` with the cause `retries exhausted` if the budget was greater than 0, or `failure` if it was 0; then `on step failure`.
4. **An abnormal termination:** `on step abnormal termination` first. Then, when `retry on abnormal termination` is on, by points 2 and 3 with the cause `abnormal termination` in place of `failure`; when it is off, `step_given_up` with the cause `abnormal termination`, then `on step failure`.
5. **Skipped:** no hook and no step decision.
6. **A step with a rescue step**, on the attempt its rescue step will fix: as today, `step_rescuing` after a failure, and `on step abnormal termination` then `step_rescuing` after an abnormal termination, unless that hook returns `FailWorkflow`.

`on step retry` returning `FailWorkflow` gives the step up with the cause `FailWorkflow`, as today; that is how a policy stops retries that the reason or the error shows to be pointless.

### Causes

1. The causes are `failure`, `abnormal termination`, `retries exhausted` and, for a step given up by a hook, `FailWorkflow`. The cause `retriable failure` no longer exists.
2. `on step retry` and `step_retrying` carry the cause `failure` or `abnormal termination`.
3. `step_rescuing` carries the cause `failure` or `abnormal termination`.
4. `step_given_up` and `on step failure` carry `failure`, `abnormal termination`, `retries exhausted` or `FailWorkflow`, as before.

### What a failure carries

The rule of proposal 0083 is unchanged; only the wording of `retries exhausted` follows the flag's removal: the reason of the last attempt, if it reported a failure, and the error of the last attempt, if it ended in an abnormal termination.

| Cause | Reason | Error |
|---|---|---|
| `failure` | the step's reason | none |
| `retries exhausted` | the reason of the last attempt, if it reported a failure | the error of the last attempt, if it ended in an abnormal termination |
| `abnormal termination` | none | the error |
| `FailWorkflow` | the reason the hook gave | none |

### Rescue steps

A rescue step still has no retry budget and no `retry on abnormal termination` setting. A rescue step that fails is given up under the ordinary rules with a budget of 0: with the cause `failure` or `abnormal termination`. The run of a rescued step after its rescue follows the same rules.

## Events

| Event | When | Carries |
|---|---|---|
| `step_failed` | the attempt reported Failure | step, attempt, reason (code, message, details) |
| `step_retrying` | the step goes to `MustRetry` and will be attempted again | step, the attempt that failed, the cause (`failure` or `abnormal termination`), decided by |
| `step_rescuing` | the step goes to `Rescuing` | step, the attempt that failed, the cause (`failure` or `abnormal termination`), the rescue step's name, decided by |

`step_failed` no longer carries `retriable`. No event is added or removed.

## Conformance

Changed scenarios stay with the cases of the proposal they belong to and gain the tag `@proposal-0106`, as FORMAT.md says. In `STEPS.md`, the scripted outcome `retriable failure` and the sentence that marks an abnormal termination retriable are removed, and a sentence turns `retry on abnormal termination` off; in `FORMAT.md`, the `retriable` column of event tables is removed.

The cases must check that:

1. a failure with budget left is retried, and `on step retry` receives the cause `failure`;
2. a failure with a budget of 0 is not retried, and the step is given up with the cause `failure`;
3. a failure that spends a budget greater than 0 is given up with the cause `retries exhausted`;
4. `on step retry` returning `FailWorkflow` stops the retries: `step_given_up` with the cause `FailWorkflow`, and `on step failure` not called;
5. by default an abnormal termination with budget left is retried, after `on step abnormal termination`, and `on step retry` receives the cause `abnormal termination`;
6. an abnormal termination that spends the budget is given up with the cause `retries exhausted`, carrying the error;
7. with `retry on abnormal termination` off, an abnormal termination is not retried even with a budget, and the step is given up with the cause `abnormal termination`;
8. `step_failed` carries the step, the attempt and the reason, and no retriable flag;
9. a step with a rescue step is rescued after a failure, and after an abnormal termination even with `retry on abnormal termination` on; `step_rescuing` carries the cause `failure` or `abnormal termination`;
10. a rescue step that reports a failure is given up with the cause `failure`.

Every existing scenario that scripts `retriable failure` scripts `failure` instead, and every scenario that relied on a failure not being retried despite a budget, or on an abnormal termination not being retried by default, states what it needs.

## Alternatives considered

- **Keep the flag as a hint for hooks.** Rejected: a hint the executor ignores is noise, and the reason's code already tells a hook what happened.
- **Remove `retry on abnormal termination` entirely,** leaving `on step abnormal termination` returning `FailWorkflow` as the only way to stop those retries. Not chosen: a step whose unexpected errors may have side effects should say so in its declaration, where a reader sees it, rather than in a policy.
- **Keep `abnormal termination retriable` off by default.** Rejected: with failures retried by budget alone, a budget would mean one thing for failures and another for abnormal terminations.
- **Report `retries exhausted` even with a budget of 0.** Rejected: nothing was retried, and the cause `failure` or `abnormal termination` says what actually ended the step.

## Open questions

None.
