# 0133: Hooks before and after a journey, a step and an attempt

- Issue: [#133](https://github.com/itinera-dev/spec/issues/133)
- Tier: 2
- Amends: [0003](0003-rescue-steps.md), [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0040](0040-events-facts-and-decisions.md), [0054](0054-how-custom-code-fails.md), [0058](0058-building-policies.md), [0085](0085-aborted-result-has-no-data.md) and [0108](0108-test-support.md)

## Summary

Six new hooks, in three pairs, that run before and after a journey, a step and an attempt: `before journey` and `after journey` on workflow policies, and `before step`, `after step`, `before attempt` and `after attempt` on step policies. Unlike today's hooks, each may be defined by several policies; the before hooks run in declaration order and the after hooks in reverse, so that pairs nest. `before step` may bypass the step, and `before journey` may fail the journey before its first step; the other four return nothing. After hooks run on every ending, an abort included, so that what a before hook opened, such as a transaction through a role, is always closed.

Part of one set with proposals 0134 (pending steps), 0130 (cancellation) and 0135 (timeouts), which use the terms this proposal defines. References such as "#134" are to the issues of those proposals; numbered references such as "6.1 point 4" are to the chapters of specification 0.1.

## Motivation

1. **Nothing runs before a step or an attempt.** After an attempt there is always at least one hook, but before it there is none, so a workflow cannot prepare around a step's work: begin a transaction, take a lock, start a timer.
2. **Settling work when things go wrong.** A step's work often has to be committed or rolled back, together with the step's outcome. Today the only places are the step itself, which then knows about the flow, or today's outcome hooks, which are defined at most once per step, are not called for a skipped attempt or one that goes to a rescue step, and never run after an abort. A sub journey would serve, but is a heavy solution.
3. **A journey cannot check its input.** A journey has no chance to check that its initial data is well formed before its first step, other than throwing, which aborts.
4. **Concerns should not collide.** A transaction, a lock and a timer are independent concerns, each in its own policy. With hooks defined at most once per step, two policies that both need to act after a step cannot both be attached to it.

## Design

### Words

1. **Ending:** how an attempt, a step or a journey ended, as an after hook receives it. An attempt's ending is succeeded, failed, abnormal termination, skipped, pending (#134), cancelled (#130) or aborted. A step's ending is succeeded, failed, skipped, bypassed, abandoned (#134), cancelled (#130) or aborted. A journey's ending is succeeded, failed, cancelled (#130) or aborted. The table in "Statuses and endings" below gives each with its status.
2. **Before hooks** and **after hooks:** the six hooks of this proposal. Today's step hooks, `on step success`, `on step failure`, `on step retry` and `on step abnormal termination`, and `on step pending` (#134), are called **outcome hooks** below.
3. **A step's turn** begins when the scan first reaches it and the journey goes on past the checkpoint `before step` (#130), whether or not any policy defines `before step`. A before hook **was called** when the executor called it, whatever it then did: returned, bypassed, returned `FailWorkflow` or failed.
4. Chapter 1's definitions of step hooks ("called in response to a step's outcome") and workflow hooks ("called when the journey ends") are widened to cover them.

### The hooks

1. **Per step:** `before step` and `after step`. **Per attempt:** `before attempt` and `after attempt`. **Workflow hooks:** `before journey` and `after journey`. Each is defined by a policy and acts through roles, like today's hooks.
2. **Several per step or per workflow.** Any number of the policies attached to a step, or to a workflow, MAY define each of the six. The rule that each hook is defined at most once (3.3 point 4, `hook defined twice` in 3.6) applies only to today's hooks.
3. **Order.** Before hooks run in declaration order. After hooks run in reverse declaration order, so that what opened first closes last.
4. **Which after hooks run.** At each level, the after hook of a policy runs when that policy's before hook at that level was called, including the before hook that bypassed, returned `FailWorkflow` or failed, and when the policy defines no before hook at that level. It does not run when its policy's before hook was never called, because an earlier one stopped or failed, nor when the policy's instance for that level was never built. Each after hook runs at most once for its level: once a step's `after step` hooks have run, nothing runs them again.
5. **When several steps close at once,** after an abort, a cancellation (#130), or a journey that ends with steps still pending (#134), the `after step` hooks not yet called of every step whose turn began run, step by step, in reverse order of when each step's turn began. A step that ended the journey by its own decision, such as being given up, has already run its `after step` hooks as part of that decision.
6. **Policy instances.** `before journey` and `after journey` use the journey's instance of each workflow policy (6.1 point 4). `before step` and `after step` use one instance of each step policy built for the step, at its turn, before `before step`, only for the policies attached to the step that define `before step` or `after step`, as their declarations say (3.3); that instance also serves the step's first attempt, so it is not built again for it, and stays until the step's `after step` hooks have run; later attempts get new instances, and every other step policy is built only for each attempt, as in tier 1; if building it fails, the journey is aborted with `policy could not be built`, naming the step and the policy, and no attempt, and the step becomes `Aborted` from `NotExecuted`, as 6.1 point 6 says for an attempt; 5.4, the abort table of 5.7 and the row of 5.10 point 2 are amended for it. `before attempt` and `after attempt` use the attempt's instances (6.1 point 5). This amends 6.1 points 3 and 5 and 3.3 point 9, under which step policies are built only for an attempt.
7. **Rescue steps** have no policies, so none of the six runs for a rescue step. When a journey is aborted while a rescue step runs, the after hooks are those of the step it rescues.

### When each runs

1. **`before journey`**: once, after `journey_started` and before the first step.
2. **`before step`**: once, at the step's turn in the scan, before anything else happens for the step: before its first `attempt_started`, and before `step_waiting` for a step that waits (#134).
3. **`before attempt`**: for each attempt, after `attempt_started` and after the attempt's step policies are built, before the step's inputs and services are resolved and before it is built.
4. **`after attempt`**: for each attempt, right after its outcome fact, and before the attempt's contributions are committed or discarded, its outcome hooks, `step_rescuing` or the pending rules of #134.
5. **`after step`**: once, when the step's status is about to become final:
   1. **succeeded:** after the last attempt's `after attempt`, before the contributions are committed and before `on step success`; the step becomes `Succeeded` only when its `after step` hooks have returned;
   2. **given up:** after `step_given_up`, before `on step failure`;
   3. **skipped:** after the attempt's `after attempt` hooks, before `contributions_discarded` and before the journey moves on;
   4. **bypassed:** after `step_bypassed`, before the journey moves on;
   5. **rescued, with a failing rescue step** (chapter 7, 7.4 point 3): after the rescue step's `step_given_up`, with the ending failed, before `after journey` and `journey_failed`;
   6. **abandoned, cancelled or aborted:** as #134, #130 and "After an abort" below say.
   So the journey never leaves a step that cannot be pending, to the next step, to its end or, in a later tier, to another workflow, before the step's `after step` hooks have run. For a step that may be pending (#134), `after step` runs when it reaches its final status, which may come after later steps have run.
6. **`after journey`**: once, when the journey's ending is known, before `on workflow success` or `on workflow failure`, and before `journey_succeeded` or `journey_failed`.

### Contributions

A step's contributions are committed once its after hooks have returned: for an attempt that succeeded, after its `after attempt` hooks and, on the step's last attempt, its `after step` hooks, and before `on step success`. A skipped attempt's contributions are discarded at the same point, after its after hooks, with `contributions_discarded`. This amends 2.8 point 3 and 4.6 point 5, which commit or discard them right after the outcome fact. A rollback in an after hook that fails therefore never leaves the data bag holding data the step's work no longer backs.

### What each returns

1. **`before journey`**: nothing, which continues, or `FailWorkflow` with a reason. The first that returns `FailWorkflow` wins, and the later ones are not called. The journey fails before its first step: every step stays `NotExecuted`, `journey_failed` is decided by that policy and hook and names no step, the `after journey` hooks run (point 4 of "The hooks"), then `on workflow failure`, and the result carries the reason. `FinishWorkflow` from `before journey` is an `invalid lifecycle`.
2. **`before step`**: nothing, which continues, or **bypass**, with an optional reason. Bypass is not a lifecycle: it acts on one step, not on the journey. The first that bypasses wins, and the later ones are not called. A bypassed step is not built: no input adapter or service provider is called for it, and it has no attempt. Its status becomes `Bypassed`, which is final, and `step_bypassed` is decided by that policy and hook. The `after step` hooks run, then the journey moves on. A lifecycle from `before step` is an `invalid lifecycle`.
3. **`before attempt`, `after attempt`, `after step` and `after journey`**: nothing. They are void, as workflow hooks are in tier 1 (6.3); what they return is never read.
4. The 6.3 table gains a row for each of the six.

### What each may request

**Every one of the six** MAY request the journey ID, data from the workflow, read from the data bag and never through an adapter (6.6 point 3), roles (6.8) and a means of emitting `journey_*` events and setting global metadata (6.7 point 6). **None** MAY request services, the data bag itself or a contributor: these hooks prepare and settle, and data reaches steps through steps, input adapters and outcome hooks.

1. **`before journey`**: nothing more.
2. **`before step`**: the step's name.
3. **`before attempt`**: the step's name and the attempt number.
4. **`after attempt`**: the step's name, the attempt number, the attempt's ending, the outcome the attempt reported, the failure's reason, the error, and the attempt's contributions (6.6 point 1).
5. **`after step`**: the step's name, the step's ending, the number of attempts made, the last reason and error, the cause it was given up for, and the last attempt's contributions.
6. **`after journey`**: the journey's ending, and the reason of the failure or the abort.

Two rules for after hooks:

1. **Optional only.** Every request of an after hook MUST be optional, except the journey ID, roles, the means of emitting, the step's name, the attempt number, the ending and the number of attempts, which always exist. Data from the workflow, the outcome reported, the reason, the error, the cause, the contributions and the abort reason may not exist: after an abort, a bypass or a failed before hook, or because of the ending. A required one is the new violation `required request in after hook`, found when the workflow is built (3.6).
2. **After an abort,** every such request is answered absent, so that what an abort interrupted stays unobservable (0085 point 5, 5.8 point 1.3). The ending, the abort reason, the journey ID, the step's name and the attempt number are still given.

### When one fails

A failing before or after hook is recorded by the new fact `after_hook_failed` or `before_hook_failed`, naming the policy, the hook, the step and attempt if any, and the error's message. The outcome fact the step reported is never replaced: each attempt still has exactly one outcome fact (2.7 point 1), and for an attempt whose `before attempt` failed, that fact is `before_hook_failed`. Each attempt has at most one step decision, but a failing after hook may give one to an attempt that succeeded, as points 2 and 5 say, which amends 2.7 point 2.

1. **`before attempt`:** the step is not built, and the attempt ends as an abnormal termination with the cause `before hook failed`, carrying the hook's error. Its outcome fact is `before_hook_failed`, its `after attempt` hooks run as point 10 says, and 6.2 point 5 then applies, with that cause in place of `abnormal termination`.
2. **`after attempt`, after a Success:** the attempt goes the abnormal termination path of 6.2 point 5, with the cause `after hook failed` and the hook's error, and its contributions are not committed. A retry is a new attempt, with a fresh `before attempt`. In points 1 and 2, a step with a rescue step is rescued, as for an abnormal termination (7.2 point 1), and `step_rescuing` carries the cause `before hook failed` or `after hook failed`; this amends 7.2 point 1 and the causes of `step_rescuing` in 2.7.
3. **`after attempt`, after any other ending:** recorded, and nothing else changes.
4. **`before step`:** the step is not built and has no attempt. It is given up: `step_given_up` with the cause `before hook failed`, naming the policy and hook and no attempt, the step goes from `NotExecuted` to `Failed`, and its `after step` hooks run. `on step failure` is not called, since the step had no attempt and so no attempt's policy instances. The journey fails with the cause `before hook failed`.
5. **`after step`, after a success:** the step is given up instead of succeeding: `step_given_up` with the cause `after hook failed`, which is the step's one decision, the step becomes `Failed`, its contributions are not committed, and `on step failure` runs.
6. **`after step`, after any other ending:** recorded, and nothing else changes.
7. **`before journey`:** the journey is aborted with `hook failed`.
8. **`after journey`, after a success:** the journey fails instead: `journey_failed` names no step, carries the cause `after hook failed` with the policy and hook, and is decided by `default`; then `on workflow failure` runs. After any other ending, recorded, and nothing else changes.
9. **The rest of the level still runs.** A failing after hook does not stop the other after hooks of its level. When several fail, the first failure's error is the one carried. The remaining hooks of the level receive the ending their level began with, even when a failure changes it, for example from succeeded to given up.
10. **Pairs close.** When a before hook fails, the after hooks named by point 4 of "The hooks" still run.
11. **Aborts from these hooks.** A request that aborts (for example `wrong type`, which aborts even for an optional request, 6.6 point 3), something that is not a value, a reserved metadata key or a failing reporter aborts the journey as for any hook (6.5 point 2). The remaining after hooks then run as "After an abort" says.
12. **The result.** For the causes `before hook failed` and `after hook failed`, the result's table in 5.8 gains rows: no reason, and the hook's error.

This amends 6.5 point 1 and 5.10 points 1 and 2, under which every hook failure aborts, and 5.10 point 3, whose list of calls into custom code gains the six hooks with these outcomes.

### After an abort

An abort means something illegal happened (1.5, 5.7 point 3). Still, what a before hook opened must be closed:

1. **After `journey_aborted`,** the after hooks run, nested: the `after attempt` hooks of the attempt that was running, then the `after step` hooks of the steps point 5 of "The hooks" names, then the `after journey` hooks; each only as point 4 of "The hooks" says, each once, in reverse declaration order. Nothing else runs.
2. **Nothing can abort again.** The first abort reason stands (2.3 point 4). An after hook that throws or cannot resolve a request is stopped, the abort reason and its details do not change, and the remaining after hooks still run.
3. **Nothing is delivered:** `journey_aborted` is the last event (2.4 point 3), so these hooks' events and global metadata reach no reporter.
4. **Their failures stay visible:** the aborted result MAY carry them as diagnostics, which change neither the abort reason nor its details.
5. An unrecoverable failure (5.10 point 5) stays outside the model: nothing runs after it.
6. A cancelled journey runs these after hooks too, with the ending `cancelled`, as #130 says.

This amends 1.5 ("No hook runs after an abort"), 2.7 point 3, 5.7 point 3, 6.4 point 4 and 6.5 point 3, and 0010's rejected alternative of workflow hooks on abort, for after hooks only.

### Bypass, the scan and the vision

Bypass amends 0009 point 10 ("Hooks cannot reorder, skip or jump between steps") and 5.5 point 1, for `before step` only. It is a flow decision about whether a step runs in this journey, taken by a policy of the workflow and recorded as a decision, and it never reorders steps or jumps between them, so it is not branching. 4.5 point 4 ("Only the step decides to skip") stays true, since a bypass is not a skip. The book should say that a bypass decides from what the workflow knows, such as configuration or what an earlier step contributed, and that a business rule belongs in a step (6.6 point 7).

Running after hooks for a skipped step amends 4.5 point 1 and 6.2 point 6 ("Skipped: no hook and no step decision"): a skipped step still has no outcome hook and no step decision, but its after hooks run.

### Statuses and endings

This table is shared by #133, #134 and #130. A step that never had its turn stays `NotExecuted` (5.4 point 2, 6.3 point 3).

| How the step ended | Its status | Its ending, for `after step` |
|---|---|---|
| it succeeded | `Succeeded` | succeeded |
| it was given up | `Failed` | failed |
| it skipped itself | `Skipped` | skipped |
| a `before step` bypassed it | `Bypassed` | bypassed |
| its rescue step failed (7.4) | `Rescuing`, as today | failed |
| the journey ended by a failure or `FinishWorkflow` while it was `Pending` or `Waiting` (#134) | `Abandoned` | abandoned |
| the journey was cancelled after its turn began and before it was final (#130) | `Abandoned` | cancelled |
| the journey was aborted after its turn began and before it was final | as today (5.4, 5.7), and `Aborted` for a `Pending` or `Waiting` step (#134) | aborted |

### Implementation ideas

A transaction through roles: a workflow instance runs at most one journey (5.1 point 5), so a role can hold the journey's current transaction. `before attempt` asks the role to begin it; an attempt-scoped provider of the step's connection, called on the same workflow instance (spec#131), joins it; and `after attempt` asks the role to commit or roll back from the attempt's ending.

## Events

1. `hook_called` is emitted for each of the six hooks that returns without failing, with its policy and hook, the step and attempt when there are any, and, for `before journey` and `before step`, what it returned; for the four void hooks, none, since what they return is never read (5.10 point 6). 2.5 points 2 and 3 and 2.9 point 2 gain a form with no attempt, for `before step`, `after step`, `step_bypassed`, `before_hook_failed` and `after_hook_failed` at the step level, and `step_given_up` when `before step` failed.
2. **New decision `step_bypassed`:** the step, the reason if any, decided by the policy and hook.
3. **New facts `before_hook_failed` and `after_hook_failed`:** the policy, the hook, the step and attempt if any, and the error's message.
4. **New status `Bypassed`**, final; 1.2's list of statuses, 5.4 point 3 (run-once), 5.5 point 2 (the scan passes over it) and 5.7 point 1 (it counts as done) are amended for it. **New transitions** in 5.4 and the 5.5 diagram: `NotExecuted` to `Bypassed`, and `NotExecuted` to `Failed` when `before step` fails.
5. **New causes** for `step_given_up` and the abnormal termination path: `before hook failed` and `after hook failed`; for `journey_failed`: `after hook failed`.
6. `journey_failed` decided by `before journey`, or caused by a failing `after journey`, names no step.
7. **Test support** (chapter 8): new path lines for `step_bypassed`, `before_hook_failed` and `after_hook_failed`, and a form of `journey failed` that names no step. 8.3 point 2 is amended, since `before_hook_failed` and `after_hook_failed` change the flow without being an outcome or a decision.
8. **Also amended:** 2.8 points 2, 4 and 5, where the six hooks and their events fall, as "When each runs" says; and 5.10 point 6, since `before journey` and `before step` are calls with a result: nothing, `FailWorkflow` or a bypass.

## Conformance

Cases, tagged with this proposal and `@tier-2`. A new runner sentence observes the order of hook calls even where no event is delivered, such as `Then the hooks were called in order:` with a table of policy and hook. They check:

1. the order of the six hooks around a step with two attempts, and of several of each, after hooks in reverse, including where commits and `on step success` fall;
2. a bypass by the second of three `before step` hooks: the third not called, the step `Bypassed` with no attempt, no adapter or provider called, and the `after step` hooks of the first two run;
3. an after-only policy: its after hooks run in every case below;
4. `before journey` returning `FailWorkflow`: no step runs, `journey_failed` decided by it and naming no step, then `after journey` and `on workflow failure`;
5. `before journey` failing: an abort with `hook failed`, and the paired `after journey` hooks;
6. `before step` failing: `step_given_up` with `before hook failed` and no attempt, `after step`, no `on step failure`, and the journey failed;
7. `before attempt` failing: `before_hook_failed` as the attempt's outcome fact, the abnormal termination path with `before hook failed`, only the after hooks that should run, and a step with a rescue step rescued with that cause;
8. `after attempt` failing after a Success: one outcome fact, `after_hook_failed`, the abnormal termination path, retried while the budget lasts, contributions not committed;
9. `after step` failing after a success: the step given up with `after hook failed`, contributions not committed, `on step failure`;
10. `after journey` failing after a success: the journey fails with `after hook failed`;
11. one of two `after attempt` hooks failing: the other still runs;
12. `after attempt` for a skipped attempt and for an attempt that goes to a rescue step, and `after step` for a step whose rescue step failed;
13. an abort during an attempt: the after hooks of every level run once each, in order, none aborts again, requests for data are answered absent, and an after-only policy whose instance was never built runs nothing;
14. an abort caused by an after hook in a normal flow;
15. what the after hooks receive: the endings and the number of attempts;
16. the violations `required request in after hook` and `invalid lifecycle` for the new hooks, and that `hook defined twice` does not apply to them;
17. how often each policy is built: once at the step's turn for a policy with step-level hooks, reused by the first attempt, and once for each later attempt.

## Alternatives considered

1. **Today's outcome hooks as the "after attempt":** defined at most once per step, not called for a skipped attempt or one that goes to a rescue step, and never run after an abort, so a transaction could be left open.
2. **A sub journey** (tier 4, after renumbering): it serves, but is heavy for preparing and settling one step.
3. **Service providers that settle what they made** when their scope ends: one rule instead of six hooks, but only for resources a provider makes.
4. **Compensation steps,** run in reverse order when a journey fails, to undo earlier steps' business effects (sagas): a different need, and possibly a later proposal.
5. **A refusal for invalid input** instead of `FailWorkflow` from `before journey`: a refusal has no events, while a rejected input is something that happened.
6. **Before hooks with a contributor:** a before hook writing data the step then reads would be a hidden second input adapter.
7. **No bypass, gating left to the step:** the step can already skip, but only after it is built and its services made; a bypass decides before anything is built.

## Open questions

None.
