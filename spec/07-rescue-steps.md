# 7. Rescue steps

Specification 0.1. Written from proposal [0003](../proposals/0003-rescue-steps.md), with the services of [0095](../proposals/0095-services-provided-to-steps.md) and the input adapters and service providers of [0098](../proposals/0098-rescue-step-adapters-and-providers.md).

A rescue step fixes the cause of a step's failure, after which the failed step runs once more. This chapter defines how one is declared, when it runs, what it receives, and how its end decides the journey. Everything not said here follows the chapters before it: a rescue step is a step.

## 7.1 Declaring a rescue step

1. **One rescue step per step, at most.** A step descriptor MAY hold a **rescue step**: how to build it, as for any step (chapter 4). That is all the workflow declares about it: the rescue step has no step descriptor of its own, no name chosen by the author, no step policies, no retry budget and no `abnormal termination retriable` setting.
2. **Its name is `<step name>_rescue`**, for example `charge_rescue` for the step `charge`. A declared step with that name is the violation `duplicate step name`. Where an input adapter or a service provider is attached to steps, that name MAY be named as if it were a step (chapter 3, 3.4 point 2 and 3.9 point 7); the rescue step stays undeclared. Attaching to `<step name>_rescue` when the step has no rescue step is the violation `input adapter for unknown step` for an adapter, and `service provider for unknown step` for a provider, as for any name the workflow does not have.
3. **No retries.** A step with a rescue step MUST have a retry budget of 0. A step descriptor holding both a rescue step and a retry budget greater than 0 is the violation `rescued step retried`, caught when the workflow is built (chapter 3, 3.7). So neither the step's first attempt, nor its rescue step, nor its run after the rescue is ever retried.
4. **Execution mode.** A rescue step is synchronous or asynchronous, like any step, and is checked for `mode not accepted` at admission with the others.
5. **The listing** shows, in the entry of each step that has a rescue step, a rescue section: the rescue step's name, the name of its input adapter, if any, and the services it requests, each with the name of the provider that serves it. The rescue section has no position and no policies (chapter 3, 3.1 point 2).
6. The same kind of step MAY rescue several steps. Each rescue is its own step in the journey, named after the step it rescues.

## 7.2 When a step fails

1. **What triggers the rescue.** When an attempt of a step that has a rescue step, and has not been rescued yet in the journey, ends in `step_failed`, retriable or not, or in `step_abnormal_termination`, whatever `abnormal termination retriable` says:
   1. after an abnormal termination, `on step abnormal termination` is called as for any step (chapter 6, 6.2 point 5). If it returns `FailWorkflow`, the step is given up as for any step: `step_given_up` with the cause `FailWorkflow`, no rescue, and the journey fails;
   2. otherwise the executor MUST record the decision `step_rescuing`, decided by `default`. `on step retry` and `on step failure` MUST NOT be called for this attempt: its failure is handed to the rescue;
   3. the step's status becomes `Rescuing`, and the executor inserts the rescue step into the journey's steps, as `NotExecuted`, immediately before the failed step;
   4. the failed attempt's contributions are not committed, as for any failed attempt (chapter 4, 4.6 point 5), but are kept for the rescue step's requests (7.3 point 2).
2. **The scan does the rest** (chapter 5, 5.5). The first step that is not finished is now the rescue step, which is built and run as its attempt 1. A `Rescuing` step is built and run as its next attempt, like a `MustRetry` step, but the scan reaches it only after its rescue step succeeded.

## 7.3 What the rescue step receives

1. **Like any step**, the rescue step receives its inputs, a contributor, a step reporter and its services (chapter 4, 4.1). It sees everything committed so far, including what the hooks of the failed attempt contributed. It gets only what is attached to its own name, `<step name>_rescue`, and, for services, the providers attached to no step, and nothing attached to the step it rescues:
   - **its inputs** are resolved as for any step (chapter 4, 4.2): through the input adapter attached to `<step name>_rescue`, if there is one, then from the data bag. The same events and aborts apply, naming the rescue step. An input adapter attached to the step it rescues is never asked for the rescue step's inputs. When the rescue step's adapter requests the name of the step being built, it receives `<step name>_rescue`;
   - **its services** come from the service provider of each key attached to `<step name>_rescue`, if there is one, and otherwise from the provider of that key attached to no step (chapter 3, 3.9 point 7).
2. **Data from the step it rescues.** A rescue step MAY also request, from the step it rescues, what a step hook may request from the step it acts on (chapter 6, 6.6 point 1): the failed attempt's contributions by key and type, the failure's reason and the error, each as required or optional, under the same rules:
   - a required request for something that does not exist MUST abort the journey with `required data missing`;
   - a value of the wrong type MUST abort the journey with `wrong type`, whether the request is required or optional;
   - an optional request for a contribution the failed attempt did not make is absent, and `optional_input_absent` is emitted, naming the rescue step and the key; an optional request for a reason or an error that does not exist is answered absent, with no event, as for a hook;
   - an abort while resolving them names the rescue step and the key, or, for the failure reason or the error, which of the two was requested.

   These requests are resolved while the rescue step is built, together with its inputs, in the order it declares them. They are never asked of the rescue step's input adapter, which serves inputs only. Like every input, what they answer is read-only (chapter 4, 4.7).

## 7.4 How the rescue ends

1. **The rescue step succeeds.** Its contributions are committed, and the executor MUST record the decision `step_rescued` for the rescued step, decided by `default`. The scan then reaches the `Rescuing` step and runs it again, as its next attempt, building new step policies as for every attempt (chapter 6, 6.1 point 5).
2. **The run after the rescue** follows the ordinary rules with a retry budget of 0: on success the journey goes on; on a failure or an abnormal termination the step is given up (`step_given_up` with `failure`, `retries exhausted` or `abnormal termination`, after `on step abnormal termination` when the attempt ended in one), its `on step failure` is called, and the journey fails by default. A step is rescued at most once per journey.
3. **The rescue step fails or ends in an abnormal termination.** It is a step that failed, under the ordinary rules with a retry budget of 0 and no policies: `step_given_up` for the rescue step, with the cause `failure`, `retries exhausted` (for a retriable failure) or `abnormal termination`, decided by `default`. The journey fails, and `journey_failed` names the rescue step with its cause, reason and error (chapter 5, 5.8). No step hook is called. The rescued step stays `Rescuing`, unless the journey is aborted afterwards (7.5), and never runs again, as the steps after a failed step stay `NotExecuted`. `on workflow failure` runs as for any failed journey.
4. **The rescue step reports Skipped.** A rescue step MUST NOT skip: it either fixes the cause or fails. A skip is a configuration error that can only show up during a journey: its status is `Skipped`, as for any step that skips (chapter 4, 4.5); its contributions are discarded and `contributions_discarded` is emitted; then the journey MUST be aborted with the abort reason `rescue skipped`, naming the rescue step. Implementations SHOULD make a rescue step that skips impossible to express (chapter 3, 3.8).

## 7.5 Statuses and aborts

1. **`Rescuing`** means that the step's last attempt failed and was handed to its rescue step, which has not succeeded yet. It is not final. A step becomes `Rescuing` from `Executing` when `step_rescuing` is recorded. From `Rescuing` it becomes `Executing` when its next attempt starts, or `Aborted` when the journey is aborted before that.
2. **The rescue step's statuses** are those of any step: `NotExecuted` from its insertion, then `Executing`, then `Succeeded`, `Failed`, `Skipped` (7.4 point 4) or `Aborted`.
3. **An abort while a step is being rescued**, for example because the rescue step cannot be built, a reporter fails on one of its events or it skips, names the rescue step in `journey_aborted`. The rescue step becomes `Aborted` if it was `Executing`, and keeps its status if it was already final. The rescued step becomes `Aborted`, since it was `Rescuing`, which is not final.
4. **An abort on `step_rescuing` or `step_rescued`**, when a reporter fails while one of them is delivered, names the rescued step, which becomes `Aborted`. The rescue step keeps its status: `NotExecuted` after `step_rescuing`, `Succeeded` after `step_rescued`.
5. **Custom code.** A rescue step's constructor, its requests and its run are a step's (chapter 5, 5.10): a constructor that fails, an input adapter that fails for one of its inputs, or a service provider that fails for one of its services, aborts the journey with `step could not be built`, and an error that escapes its run is an abnormal termination of the rescue step (7.4 point 3).

## 7.6 Events

1. **`step_rescuing`**, a decision: the step will be rescued. It carries the step, the attempt that failed, the cause (`failure`, `retriable failure` or `abnormal termination`), the rescue step's name, and decided by. It is the step decision that follows that attempt (chapter 2, 2.7 point 2), and comes after `on step abnormal termination` when that hook is called.
2. **`step_rescued`**, a decision: the rescue succeeded and the step will run again. It carries the step, the attempt that failed, the rescue step's name, and decided by. It comes after the rescue step's `step_succeeded` and its `contribution_committed` events.
3. **The rescue step's own events** are those of any step, carrying its own name and attempt 1: `attempt_started`, its input events, its service events, its `step_*` events, its outcome fact, its committed contributions, and `step_given_up` when it fails.

A rescue that works reads, in order:

1. the step's `attempt_started` (attempt 1), its events and its outcome fact;
2. `on step abnormal termination`, if the attempt ended in an abnormal termination and a policy defines it;
3. `step_rescuing`;
4. the rescue step's `attempt_started` (attempt 1), its input events, its service events, its own events, `step_succeeded` and its `contribution_committed` events, each followed by `data_overwritten` when it replaced a value;
5. `step_rescued`;
6. the step's `attempt_started` (attempt 2), and the rest of the journey.

## 7.7 Checked by

In the conformance repository: [`0003-rescue-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0003-rescue-steps), and [`0095-services-provided-to-steps/rescue-steps.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0095-services-provided-to-steps/rescue-steps.feature). The input adapters and service providers attached to a rescue step are checked by the scenarios tagged `@proposal-0098` in [`0003-rescue-steps/declaring.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0003-rescue-steps/declaring.feature), [`0003-rescue-steps/input-adapters.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0003-rescue-steps/input-adapters.feature) and [`0095-services-provided-to-steps/rescue-steps.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0095-services-provided-to-steps/rescue-steps.feature).
