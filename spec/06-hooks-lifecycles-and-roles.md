# 6. Hooks, lifecycles and workflow roles

Specification 0.1. Written from proposals [0010](../proposals/0010-hooks-lifecycles-and-roles.md), [0002](../proposals/0002-foundations.md) as amended by [0027](../proposals/0027-received-data-read-only.md), [0008](../proposals/0008-steps.md) and [0024](../proposals/0024-abnormal-termination-retriable.md).

Which hooks the executor calls and in what order, what each may return, what hooks can read and write, and how policies reach a workflow through roles. How policies and hooks are declared and attached is defined in chapter 3.

## 6.1 The hooks

1. **Step hooks:** `on step success`, `on step failure`, `on step retry` and `on step abnormal termination`. They act on one attempt of the step whose policies define them.
2. **Workflow hooks:** `on workflow success` and `on workflow failure`.
3. Each hook is defined at most once per step or per workflow (chapter 3). A hook that no attached policy defines is not called, and its default applies.

## 6.2 Which step hooks run, and in what order

After each attempt's outcome, the executor MUST call:

1. **Success:** `on step success`.
2. **A failure that is not retriable:** `on step failure`, with the cause `failure`.
3. **A retriable failure, with retry budget left:** `on step retry`, with the cause `retriable failure`, then the next attempt.
4. **A retriable failure, with the budget exhausted:** only `on step failure`, with the cause `retries exhausted`. The retry hook MUST NOT be called when no retry is possible.
5. **An abnormal termination:** `on step abnormal termination` first. Then:
   - if the step descriptor does not set `abnormal termination retriable`, `on step failure`, with the cause `abnormal termination`;
   - if it does, by the rules of points 3 and 4: `on step retry` with the cause `abnormal termination` while the budget lasts, and otherwise `on step failure` with the cause `retries exhausted`.
6. **Skipped:** no hook (chapter 4).

## 6.3 What each hook may return

1. **A hook returns a lifecycle, or nothing.** Nothing means the default. The tier 1 lifecycles are `FailWorkflow` and `FinishWorkflow`.

   | Hook | Nothing means | May return |
   |---|---|---|
   | `on step success` | the next step, or success after the last | `FinishWorkflow`, `FailWorkflow` |
   | `on step failure` | the journey fails | `FailWorkflow` |
   | `on step retry` | the step is tried again | `FailWorkflow` |
   | `on step abnormal termination` | continue as 6.2 point 5 says | `FailWorkflow` |
   | `on workflow success` | nothing to decide | no lifecycle in tier 1 |
   | `on workflow failure` | nothing to decide | no lifecycle in tier 1 |

2. **A lifecycle a hook may not return** MUST abort the journey with the abort reason `invalid lifecycle`. Implementations SHOULD make it impossible to express where their language allows it.
3. **`FinishWorkflow`** ends the journey at once as succeeded. The steps that did not run remain `NotExecuted`.
4. **`FailWorkflow`** ends the journey at once as failed, and carries a reason: a code, an optional message and optional details. The result MUST name the step whose hook returned it, and that reason. No further step hook is called for that step.
   - From `on step success`, the step stays `Succeeded`, and its contributions are committed.
   - From `on step retry` or `on step abnormal termination`, the step becomes `Failed`, and `on step failure` MUST NOT be called.
   - From `on step failure`, the step is `Failed`, and the result carries the lifecycle's reason.

## 6.4 Workflow hooks

1. `on workflow success` MUST run once when the journey succeeds, including through `FinishWorkflow`.
2. `on workflow failure` MUST run once when the journey fails, including through `FailWorkflow`.
3. Both run at the end, before the result is produced and before `journey_succeeded` or `journey_failed`.
4. Neither runs when the journey is aborted.

## 6.5 Hooks never throw

1. A hook that throws or panics MUST abort the journey with the abort reason `hook threw`.
2. After an abort, no hook runs.

## 6.6 What hooks can read

1. **Data from the step.** A step hook MAY request, from the step it acts on:
   - its contributions, by key and type, as required or optional, including contributions not committed;
   - the failure's reason;
   - the cause of the call, for `on step failure` and `on step retry`;
   - the error, for an abnormal termination.

   Data from the step MUST be exactly what that step contributed under the key; there is no adapter in this direction. If the step did not contribute the key, a required request MUST abort the journey with `required data missing`, and an optional request is answered with the data absent. A value of the wrong type MUST abort the journey with `wrong type`, whether the request is required or optional.
2. **The step's name, the attempt number and the journey ID.** A step hook MAY request the name of the step it acts on and the attempt number. Every hook MAY request the journey ID.
3. **Data from the workflow**, exactly as steps request it: by key and type, required or optional, with the rules of chapter 4 for missing values and wrong types.
   - A step hook's request MUST be resolved as an input of the step it acts on would be: through the workflow's input adapter for that step and key if there is one, otherwise from the data bag.
   - A workflow hook's request MUST be resolved from the data bag.
   - An input adapter that fails while resolving a hook's request MUST abort the journey with the abort reason `hook threw`.
4. A hook MUST NOT be given the data bag itself.
5. **Received data is read-only.** Everything a hook receives (data from the step, data from the workflow, a reason, an error, the journey ID) is read-only for it. Changing it, where the language allows it at all, MUST NOT change the data bag, any contribution, or what any other step or hook receives.
6. Business decisions belong in steps: hooks branch on what steps reported and contributed.

## 6.7 What hooks can write

1. A hook MAY request a **contributor**, as a step does. It never has direct access to the data bag.
2. A hook's contributions MUST be committed as soon as it returns without throwing, whatever the step's outcome. A value is captured when it is contributed, and the last value for a key wins.
3. A workflow hook's contributions are committed before the result is produced, so they are part of the journey's output.
4. Overwriting a key is allowed and emits `data_overwritten`. `contribution_committed` and `data_overwritten` MUST say whether a contribution came from a hook, naming its policy and hook, or from a step.

## 6.8 Workflow roles

1. A **role** is an interface a workflow can provide: operations the workflow's own code implements, for example "send a notification". A workflow MAY provide any number of roles.
2. **Roles carry operations, not data.** A hook that needs data requests it as data from the workflow (6.6). Implementations MAY give every role a common base interface, without data access.
3. **A hook requests each role it needs as a separate declared parameter**, the same way it declares the data it requests, and MAY request several. It reaches the workflow instance through those roles only, never as a whole, so a policy can be attached to any workflow that provides the roles it needs.
4. A role MUST NOT expose the data bag, the executor or the steps.
5. Step policies and workflow policies MAY request roles; steps MUST NOT.
6. A policy attached to a workflow that does not provide a role it requests is the violation `role not provided` (chapter 3). Implementations SHOULD detect it at compile time where their language allows it.
7. **Role calls are the workflow's business.** The executor does not record them; a hook that wants them in the event stream emits a `journey_*` event itself.

## 6.9 Execution modes

Hooks MAY be synchronous or asynchronous, under the same executor rules as steps (chapter 5).

## 6.10 Checked by

In the conformance repository: [`0010-hooks-lifecycles-and-roles/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0010-hooks-lifecycles-and-roles), [`0002-foundations/data-from-the-step.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/data-from-the-step.feature), and the hook cases of [`0008-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0008-steps), [`0011-events/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0011-events), [`0024-abnormal-termination-retriable/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0024-abnormal-termination-retriable) and [`0027-received-data-read-only/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0027-received-data-read-only).
