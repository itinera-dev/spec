# 0009: Running a workflow

- Issue: [#9](https://github.com/itinera-dev/spec/issues/9)
- Tier: 1
- Amended by [#32](https://github.com/itinera-dev/spec/issues/32): configuration errors found while running.
- Amended by [#49](https://github.com/itinera-dev/spec/issues/49): custom code that throws.
- Amended by [#56](https://github.com/itinera-dev/spec/issues/56): everything in the data bag is a value.
- Amended by [#61](https://github.com/itinera-dev/spec/issues/61): the workflow instance carries its journey ID and its reporters.
- Amended by [#62](https://github.com/itinera-dev/spec/issues/62): workflow descriptors, and executors independent of how workflows are written.

## Summary

How an executor runs one workflow: the workflow instance and its data, journey IDs, step descriptors, the order of steps and the scan, step statuses, retries, and the journey result. Builds on proposals 0002 (Foundations), 0008 (Steps) and 0024.

## Motivation

Workflow authors need to know exactly what the executor does with a workflow: where the data comes from, in which order steps run, when a step is tried again, and what they get back at the end. Operators need a result that says what happened and, when something went wrong, where and why. And every setting that changes this must be visible in the workflow's own declaration, step by step, with no hidden defaults.

## Design

### The workflow instance

1. **One workflow instance per journey.** A workflow's definition is fixed and can be listed without running anything (proposal 0002). A **workflow instance** is created by the developer's own code for each journey and handed to an executor, which runs it. A failure while creating the instance happens before the executor is involved.
2. **Initial data.** The instance MAY be created with initial data, and MAY obtain more while it is created. Whatever it holds when handed to the executor MUST be the initial content of the data bag. Keys are strings; values MAY be any value the language can hold, and a wrong type is detected when a value is read.
3. **The journey started event** MUST list the initial keys and MUST NOT carry their values.
4. **The executor asks the instance** for its step descriptors, its data bag and everything else it needs. The data bag and all other journey state belong to the instance.

### Journey IDs

5. **One ID per journey.** In tier 1 a journey has exactly one run, so it has one ID, the journey ID.
6. **The workflow provides it.** If the workflow defines a custom ID generator, the executor MUST call it; otherwise the workflow's default generator MUST produce a UUID v4. A custom generator belongs to the instance, so it MAY derive the ID from the instance's own data. The caller does not pass an ID to the executor.

### Step descriptors

7. **A workflow provides one step descriptor per step**, in order. The executor builds and runs steps from them. A step descriptor holds:
   - the step's name (unique and case-sensitive, proposal 0008);
   - how to build the step;
   - its retry budget;
   - the `abnormal termination retriable` setting (proposal 0024);
   - the step policies and input adapters attached to it (proposal 0002).
8. **The retry budget is per step, explicit, and 0 by default**: the number of retries allowed after the first attempt. There is no workflow-wide default for the retry budget or for `abnormal termination retriable`; each step states its own.
9. **Attempts are counted separately from retries.** The first execution of a step is attempt 1; a step with a retry budget of N MUST be attempted at most N + 1 times.

### Order and the scan

10. **Strict order.** Steps MUST run in the order of their descriptors. Hooks cannot reorder, skip or jump between steps.
11. **The scan.** The executor repeatedly considers the first step that is not finished:
    - `Succeeded` and `Skipped` steps are passed over;
    - a `NotExecuted` or `MustRetry` step is built and run;
    - a `Failed` step ends the journey through its failure hook;
    - when no step is left, the journey succeeds.
12. **Step statuses:** `NotExecuted`, `Executing`, `MustRetry`, `Succeeded`, `Failed`, `Skipped` and `Aborted`. `Aborted` marks the step during which the journey was aborted; the steps after it remain `NotExecuted`.
13. **Run-once.** A step that ended as `Succeeded`, `Failed` or `Skipped` MUST NOT run again in the journey.

### Retries

14. A retriable failure, or an abnormal termination that the descriptor marks retriable, MUST put the step in `MustRetry` while its retry budget allows another attempt; both use the same budget. When the budget is exhausted, the step becomes `Failed` and its failure hook is called with the cause `retries exhausted`. An abnormal termination that is not retriable makes the step `Failed` at once, with the cause `abnormal termination` (proposal 0024).
15. **A retry happens immediately.** The executor MUST NOT wait between attempts in tier 1. A step that needs to wait before reporting a failure does so itself.

### The data bag

16. **One data bag per journey**, held by the workflow instance. It starts with the initial data, receives committed contributions (proposal 0008), and is read by input adapters by default (proposal 0002). Overwriting a key, including one from the initial data, is allowed and recorded by an event (proposal 0008).

### The journey result

17. **The data bag is the journey's output.** The result MUST carry the data bag as it stood at the end: the initial data plus every committed contribution.
18. **The result MUST also carry:**
    - the final status: succeeded, failed or aborted;
    - every step's status and attempt count;
    - for a failed journey, the step that failed and its failure reason or cause;
    - for an aborted journey, the step during which it was aborted, if any, and the abort reason, one of: `step could not be built`, `required data missing`, `wrong type`, `hook threw`, `invalid lifecycle`, `reporter threw`.
19. **Finishing early.** A journey ended early by the `FinishWorkflow` lifecycle succeeds; the steps that did not run remain `NotExecuted`.

## Events

This proposal adds these engine events, whose fields and place in the catalogue are defined with the event model:

- the journey started, with the initial keys;
- an attempt of a step started, with its attempt number.

## Conformance

The conformance cases for this proposal must check that:

1. the initial data is the data bag's initial content, and the journey started event lists the keys without their values;
2. a contribution may overwrite a key from the initial data;
3. without a custom generator the journey ID is a UUID v4, and with one it is the generator's value;
4. steps run in the order of their descriptors;
5. a contribution from one step is an input to a later step;
6. a failed step ends the journey, and later steps do not run;
7. without a stated budget a retriable failure is not retried;
8. a step with a budget of N is attempted at most N + 1 times, counting attempts from 1;
9. a successful journey's data bag holds the initial data and every committed contribution;
10. a failed journey's result names the failed step and its reason, and keeps the data bag as it stood;
11. an aborted journey's result names the step, with status `Aborted`, and the abort reason, and later steps are not executed.

Passing over skipped steps, run-once and abnormal terminations are checked by the cases of proposals 0008 and 0024. The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0009-running-a-workflow/`.

## Alternatives considered

- **Callers passing the journey ID.** Rejected: the workflow owns its identity; a custom generator can derive the ID from the instance's data.
- **Workflow-wide defaults for step settings.** Rejected: every step states its own retry budget and its `abnormal termination retriable` setting, so the declaration shows everything.
- **An "abort on abnormal termination" setting.** Rejected (proposal 0024): an abnormal termination is a failure of the step, never an abort.
- **Delays between retries in tier 1.** Deferred: a delay or a backoff will be a later parameter of the step descriptor, available to every executor.

## Open questions

- Delays and backoff between attempts.
- Several runs in one journey, and separate run IDs (tier 2).
