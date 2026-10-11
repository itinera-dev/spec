# 0003: Rescue steps

- Issue: [#3](https://github.com/itinera-dev/spec/issues/3)
- Tier: 1
- Amends: [0002](0002-foundations.md), [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0032](0032-configuration-errors.md), [0040](0040-events-facts-and-decisions.md), [0042](0042-configuration-errors-before-the-journey.md) and [0054](0054-how-custom-code-fails.md)

## Summary

A step may declare a **rescue step**: a step that fixes the cause of a failure. When the step fails, by reporting a failure or by ending in an abnormal termination, the executor inserts the rescue step just before it and the scan runs it. If the rescue step succeeds, the failed step runs once more; if that run fails, the journey fails. If the rescue step fails, the journey fails. A step with a rescue step is never retried.

It amends proposal 0002 (what a step may access), 0008 (a skipped step leaves the flow), 0009 (strict order, attempts and step statuses), 0010 (which step hooks run), 0011 and 0040 (the engine catalogue), 0032 (abort reasons), 0042 (violations caught at admission) and 0054 (rules a language may make impossible to express). It adds chapter 7, Rescue steps, and changes 1.2, 1.3, 1.4, 2.7, 2.8, 2.9, 3.1, 3.2, 3.6, 3.8, 4.1, 4.5, 5.4 to 5.7, 5.10, 6.2, 6.3 and 6.6.

## Motivation

Retry and rescue mean different things. A retry does the same thing again, in case the problem was temporary. A rescue fixes the cause first and then tries again: refresh an expired token, create a missing customer record, top up a quota.

Without rescue steps, such fixes end up inside the business step, which then knows about recovery strategy, or are impossible, because a step that failed ends the journey: a hook can only fail or finish it.

## Design

### Declaring a rescue step

1. **One rescue step per step, at most.** A step descriptor MAY hold a **rescue step**: how to build it, as for any step (chapter 4). That is all the workflow declares about it: the rescue step has no descriptor of its own, no name chosen by the author, no step policies, no input adapter, no retry budget and no `abnormal termination retriable` setting.
2. **Its name is `<step name>_rescue`**, for example `charge_rescue` for the step `charge`. A declared step with that name is the violation `duplicate step name`. An input adapter attached to that name is the violation `input adapter for unknown step`, since the workflow declares no such step.
3. **No retries.** A step with a rescue step MUST have a retry budget of 0. A step descriptor holding both a rescue step and a retry budget greater than 0 is a new violation, **`rescued step retried`**, caught when the workflow is built (3.7). So neither the step's first attempt, nor its rescue step, nor its run after the rescue is ever retried. A retry is any attempt after the first, except the attempt that follows a rescue step.
4. **Execution mode.** The rescue step is synchronous or asynchronous, like any step, and is checked for `mode not accepted` at admission with the others.
5. **The listing** shows, for each step, the name of its rescue step, if it has one.
6. The same kind of step MAY rescue several steps. Each rescue is its own step in the journey, named after the step it rescues.

### When a step fails

7. **What triggers the rescue.** When an attempt of a step that has a rescue step, and has not been rescued yet in the journey, ends in `step_failed`, retriable or not, or in `step_abnormal_termination`, whatever `abnormal termination retriable` says:
   1. after an abnormal termination, `on step abnormal termination` is called as today (6.2 point 5). If it returns `FailWorkflow`, the step is given up as today: `step_given_up` with the cause `FailWorkflow`, no rescue, and the journey fails;
   2. otherwise the executor MUST record the decision **`step_rescuing`**, decided by `default`. `on step retry` and `on step failure` MUST NOT be called for this attempt: its failure is handed to the rescue;
   3. the step's status becomes **`Rescuing`**, and the executor inserts the rescue step into the journey's steps, as `NotExecuted`, immediately before the failed step;
   4. the failed attempt's contributions are not committed, as for any failed attempt (4.6 point 5), but are kept for the rescue step's requests (point 9).
8. **The scan does the rest.** The first step that is not finished is now the rescue step, which is built and run as attempt 1. A `Rescuing` step is built and run as its next attempt, like a `MustRetry` step, but the scan reaches it only after the rescue step succeeded.

### What the rescue step receives

9. **Like any step**, the rescue step receives its inputs, a contributor and a step reporter. Its inputs are read from the data bag; no input adapter is used. It sees everything committed so far, including what the hooks of the failed attempt contributed.
10. **Data from the step it rescues.** The rescue step MAY also request, from the step it rescues, what a step hook may request from the step it acts on (6.6 point 1): the failed attempt's contributions by key and type, the failure's reason and the error, each as required or optional, with the same rules: a required request for something that does not exist aborts the journey with `required data missing`, a value of the wrong type aborts it with `wrong type`, and an optional request for a contribution the failed attempt did not make is absent and emits `optional_input_absent`, naming the rescue step and the key, while an optional request for a reason or an error that does not exist is answered absent with no event, as for a hook. An abort while resolving them names the rescue step and the key, or, for the failure reason or the error, which of the two was requested. They are resolved while the rescue step is built, in the order it declares its requests, and are read-only (4.7).

### How the rescue ends

11. **The rescue step succeeds.** Its contributions are committed, and the executor records the decision **`step_rescued`** for the rescued step, decided by `default`. The scan then reaches the `Rescuing` step and runs it again, as its next attempt, building new step policies as for every attempt (6.1 point 5).
12. **The run after the rescue** follows the ordinary rules with a retry budget of 0: on success the journey goes on; on a failure or an abnormal termination the step is given up (`step_given_up` with `failure`, `retries exhausted` or `abnormal termination`, after `on step abnormal termination` when there is one), its `on step failure` is called, and the journey fails by default. A step is rescued at most once per journey.
13. **The rescue step fails or ends in an abnormal termination.** It is a step that failed, under the ordinary rules with a retry budget of 0 and no policies: `step_given_up` for the rescue step, with the cause `failure`, `retries exhausted` (for a retriable failure) or `abnormal termination`, decided by `default`, and the journey fails, `journey_failed` naming the rescue step with its cause, reason and error (5.8). No step hook is called. The rescued step stays `Rescuing`, unless the journey is aborted afterwards, and never runs again, as the steps after a failed step stay `NotExecuted`. `on workflow failure` runs as for any failed journey.
14. **The rescue step reports Skipped.** A rescue step MUST NOT skip: it either fixes the cause or fails. A skip is a configuration error that can only show up during a journey: its status is `Skipped`, as for any step that skips, its contributions are discarded with `contributions_discarded`, and then the journey MUST be aborted with a new abort reason, **`rescue skipped`**, naming the rescue step. Implementations SHOULD make a rescue step that skips impossible to express (3.8).

### Statuses and aborts

15. **A new status, `Rescuing`:** the step's last attempt failed and was handed to its rescue step, which has not succeeded yet. It is not final. From `Executing` a step becomes `Rescuing` when `step_rescuing` is recorded. From `Rescuing` it becomes `Executing` when its next attempt starts, or `Aborted` when the journey is aborted before that.
16. **The rescue step's statuses** are those of any step: `NotExecuted` from its insertion, then `Executing`, then `Succeeded`, `Failed`, `Skipped` (point 14) or `Aborted`.
17. **An abort while a step is being rescued**, for example because the rescue step cannot be built, a reporter fails on one of its events or it skips, names the rescue step in `journey_aborted`. The rescue step becomes `Aborted` if it was `Executing`, and keeps its status if it was already final. The rescued step becomes `Aborted`, since it was `Rescuing`, which is not final. An abort on `step_rescuing` or `step_rescued`, when a reporter fails while one of them is delivered, names the rescued step, which becomes `Aborted`; the rescue step keeps its status.

### Custom code that fails (5.10)

18. The rescue step's constructor, its requests and its run are a step's: a constructor that fails aborts the journey with `step could not be built`, and an error that escapes its run is an abnormal termination of the rescue step (point 13).

## Events

1. **`step_rescuing`**, a decision: the step will be rescued. It carries the step, the attempt that failed, the cause (`failure`, `retriable failure` or `abnormal termination`), the rescue step's name, and decided by. It is the step decision that follows that attempt (2.7 point 2 gains it beside `step_retrying` and `step_given_up`), and comes after `on step abnormal termination` when that hook is called.
2. **`step_rescued`**, a decision: the rescue succeeded and the step will run again. It carries the step, the attempt that failed, the rescue step's name, and decided by. It comes after the rescue step's `step_succeeded` and its `contribution_committed` events.
3. **The rescue step's own events** are those of any step, carrying its own name and attempt 1: `attempt_started`, its input events, its `step_*` events, its outcome fact, its committed contributions, and `step_given_up` when it fails.
4. **`journey_aborted`** gains the abort reason `rescue skipped`.

So a rescue that works reads: the step's `attempt_started` (attempt 1), its outcome fact, `on step abnormal termination` if any, `step_rescuing`; the rescue step's `attempt_started` (attempt 1), its events, `step_succeeded`, its `contribution_committed` events, then `step_rescued` for the step; then the step's `attempt_started` (attempt 2) and the rest of the journey.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0003-rescue-steps/`. They must check that:

1. a step that reports a failure is rescued: `step_rescuing`, the rescue step's events under `<step>_rescue` with attempt 1, `step_rescued`, then the step's attempt 2, and the journey succeeds;
2. a retriable failure and an abnormal termination are rescued the same way, even when `abnormal termination retriable` is set;
3. the rescue step's contributions reach the step's second attempt;
4. the rescue step receives the failed attempt's contributions, its reason and its error, and a required one that does not exist aborts with `required data missing`;
5. the failed attempt's contributions are not committed;
6. `on step retry` and `on step failure` are not called for the rescued failure, and `on step abnormal termination` is;
7. `on step abnormal termination` returning `FailWorkflow` prevents the rescue;
8. a rescue step that fails, or ends in an abnormal termination, fails the journey: `step_given_up` for the rescue step, `journey_failed` naming it, no step hook, the step not run again, and `on workflow failure` called;
9. a step that fails again after its rescue is given up, `on step failure` is called, the journey fails, and no second rescue runs;
10. a rescue step that skips ends `Skipped`, and aborts the journey with `rescue skipped`, tagged `@rescue-skip`;
11. an abort while the rescue step is built or runs names the rescue step, and the rescued step ends `Aborted`;
12. a step with a rescue step and a retry budget is refused with `rescued step retried`, and a declared step named `<step>_rescue` with `duplicate step name`;
13. the listing shows each step's rescue step.

## Alternatives considered

- **Rescue after the retries**, with or without a new budget for the run after it. Rejected for now: one failure path per step is easier to read in a workflow and in its events. Allowing both later is a compatible amendment; forbidding them later would not be.
- **A rescue step with a descriptor of its own**, declared like any step, with policies. Rejected: the rescue belongs to the step it rescues, and the executor builds what it needs.
- **The rescued step's `on step failure` called when the rescue fails**, with a new cause `rescue failed`. Rejected: the rescue step is the step that failed, and the journey fails because of it.
- **A skipped rescue step treated as a success, or as a failure.** Rejected: a rescue either fixes the cause or fails; skipping has no meaning for it, so it is illegal.
- **A rescue step reading only the data bag.** Rejected: the failed attempt's reason, error and contributions are never committed, so the rescue could not know why the step failed.
- **Running the rescue as a hook.** Rejected: a fix is business work, done by a step, and hooks only decide flow.

## Open questions

None.
