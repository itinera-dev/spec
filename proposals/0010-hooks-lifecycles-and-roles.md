# 0010: Hooks, lifecycles and workflow roles

- Issue: [#10](https://github.com/itinera-dev/spec/issues/10)
- Tier: 1
- Amended by [#41](https://github.com/itinera-dev/spec/issues/41): hooks read data from the workflow without input adapters.

## Summary

Which hooks run and in what order, what each may return, what hooks can read and write, and how reusable policies reach the workflow through roles. Builds on proposals 0002 (Foundations), 0008 (Steps), 0009 (Running a workflow), 0024 and 0027.

## Motivation

Hooks are where flow decisions live, separated from business code. For that to work, workflow authors need to know exactly when each hook runs, what it may decide, and what it can see and change. Policies should also be reusable across workflows: a notification or audit policy should not be written once per workflow. That needs a way for a policy to reach what it requires from a workflow without depending on the workflow itself.

## Design

### Which hooks run, and in what order

1. **Step hooks** are `on step success`, `on step failure`, `on step retry` and `on step abnormal termination`. **Workflow hooks** are `on workflow success` and `on workflow failure`. Hooks are defined in policies (proposal 0002).
2. **For each outcome**, the executor MUST call:
   - **success:** `on step success`;
   - **a failure that is not retriable:** `on step failure`, with the cause `failure`;
   - **a retriable failure with retry budget left:** `on step retry`, with the cause `retriable failure`, then the next attempt;
   - **a retriable failure with the budget exhausted:** only `on step failure`, with the cause `retries exhausted`. The retry hook MUST NOT be called when no retry is possible;
   - **an abnormal termination:** `on step abnormal termination` first, then the retry hook or the failure hook by the same rules, with the cause `abnormal termination` or `retries exhausted` (proposal 0024);
   - **skipped:** no hook (proposal 0008).

### What each hook may return

3. **A hook returns a lifecycle, or nothing.** Nothing means the executor's default. The tier 1 lifecycles are `FailWorkflow` and `FinishWorkflow`. A lifecycle a hook may not return MUST abort the journey, with the abort reason `invalid lifecycle`; implementations SHOULD make it impossible to express where their language allows it.

   | Hook | Nothing means | May return |
   |---|---|---|
   | `on step success` | the next step, or success after the last | `FinishWorkflow`, `FailWorkflow` |
   | `on step failure` | the journey fails | `FailWorkflow` |
   | `on step retry` | the step is retried | `FailWorkflow` |
   | `on step abnormal termination` | continue as proposal 0024 says | `FailWorkflow` |
   | `on workflow success`, `on workflow failure` | nothing to decide | no lifecycle in tier 1 |

4. **`FinishWorkflow`** ends the journey at once as succeeded; the steps that did not run remain `NotExecuted`.
5. **`FailWorkflow`** ends the journey at once as failed and carries a reason: a code, an optional message and optional details. The journey's result MUST name the step whose hook returned it, and that reason.
   - From a success hook, the step stays `Succeeded` and its contributions are committed.
   - From a retry hook, the journey gives up at once and the failure hook MUST NOT be called.
6. **Workflow hooks** have no lifecycle in tier 1; later tiers will add some.

### Workflow hooks

7. `on workflow success` MUST run once when the journey succeeds, including through `FinishWorkflow`; `on workflow failure` MUST run once when it fails. Both run at the end, before the result is produced. Neither MUST run when the journey is aborted.

### Hooks that throw

8. **Hooks never throw.** A hook that throws or panics MUST abort the journey, with the abort reason `hook threw`. After an abort, no hook runs.

### What hooks can read

9. **Data from the step.** A step hook MAY request, from the step it acts on, its contributions (including uncommitted ones), the failure's reason, the cause of the call and the error of an abnormal termination (proposals 0002 and 0008).
10. **Attempt number and journey ID.** A step hook MAY request the attempt number; every hook MAY request the journey ID.
11. **Data from the workflow, exactly as steps request it.** A hook MAY request data from the workflow by key and type, required or optional, resolved with the same rules as a step's inputs: a missing required value or a wrong type aborts the journey, and a missing optional value is absent. A hook MUST NOT be given the data bag itself.
    - A step hook's request MUST be resolved as an input of the step it acts on would be: through the workflow's input adapter for that step and key if there is one, otherwise from the data bag.
    - A workflow hook's request MUST be resolved from the data bag.
12. **Everything a hook receives is read-only** for it (proposal 0027).
13. **Business decisions belong in steps.** Hooks branch on what steps reported and contributed.

### What hooks can write

14. **A hook MAY request a contributor**, as a step does; it never has direct access to the data bag.
15. **A hook's contributions MUST be committed as soon as it returns without throwing**, whatever the step's outcome. A workflow hook's contributions MUST be committed before the result is produced, so they are part of the journey's output. Overwriting is allowed and recorded (proposal 0008), and the events MUST say whether a contribution came from a hook or a step.

### Workflow roles

16. **A role** is an interface a workflow can provide: operations the workflow's own code implements, for example "send a notification". A workflow MAY provide any number of roles.
17. **Roles carry operations, not data.** A hook that needs data requests it as data from the workflow (item 11). Implementations MAY give every role a common base interface, without data access.
18. **A hook requests each role it needs as a separate declared parameter**, the same way it declares the data it requests, and MAY request several. It receives the workflow instance through those roles only, never the workflow as a whole, so a policy can be attached to any workflow that provides the roles it needs.
19. **A role never exposes** the data bag, the executor or the steps.
20. **Step policies and workflow policies** MAY request roles; steps MUST NOT (proposal 0002, premise 1).
21. **Admission.** A policy attached to a workflow that does not provide one of the roles it requests MUST be refused at admission, reported with every other violation; implementations SHOULD detect it at compile time where their language allows it.
22. **Role calls are the workflow's business.** The executor does not record them; a hook that wants them in the event stream emits a `journey_*` event itself.

### Synchronous and asynchronous hooks

23. Hooks MAY be synchronous or asynchronous, under the same executor capability rules as steps.

## Events

This proposal adds no engine event of its own. It requires that the events of committed contributions say whether a hook or a step made them, and that the events for aborts carry the abort reasons `invalid lifecycle` and `hook threw`. The event model defines their fields.

## Conformance

The conformance cases for this proposal must check that:

1. the retry hook runs only while a retry is possible, and then only the failure hook runs;
2. the abnormal termination hook runs before the failure hook;
3. `FinishWorkflow` from a success hook ends the journey as succeeded;
4. `FailWorkflow` from a success hook fails the journey, the step stays succeeded, its contributions are committed, and the result names the step and the reason;
5. `FailWorkflow` from a retry hook gives up at once, without the failure hook;
6. a lifecycle a hook may not return aborts the journey;
7. the workflow success and failure hooks run once, and neither runs on an abort;
8. workflow hooks receive data from the workflow, resolved from the data bag;
9. a hook that throws aborts the journey, and no hook runs afterwards;
10. hooks can request the attempt number and the journey ID;
11. a step hook's data from the workflow is resolved through its step's input adapter;
12. a hook's contributions are committed when it returns, even when the step failed, and are recorded as the hook's;
13. a workflow hook's contributions are part of the output;
14. a hook calls an operation of a role the workflow provides;
15. a policy needing a role the workflow does not provide is refused at admission;
16. the same policy works in two workflows that provide its role.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0010-hooks-lifecycles-and-roles/`.

## Alternatives considered

- **Hooks reading the data bag directly, or through a base role with read access.** Rejected: hooks request data as steps do, so they depend on keys and types, never on how the workflow stores its data.
- **Roles carrying data.** Rejected for the same reason: roles carry operations only.
- **Calling the retry hook when no retry is possible.** Rejected: the failure hook alone handles a step that cannot be tried again.
- **`FinishWorkflow` from a failure hook.** Rejected: a failed step cannot finish the journey as succeeded.
- **Workflow hooks on abort.** Rejected: an abort means something illegal happened, and no hook runs after it.
- **Steps using roles.** Rejected: steps stay isolated from the workflow.

## Open questions

- Lifecycles for workflow hooks, transfers, switches and sub workflows, in later tiers.
- Input adapters for workflow hooks, which would extend proposal 0002.
- A hook that runs before a step and may decide it does not run, in a later tier.
- Policy events, recording which policy caused an event: the events proposal.
