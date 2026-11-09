# 0135: Journey and attempt timeouts

- Issue: [#135](https://github.com/itinera-dev/spec/issues/135)
- Tier: 2
- Amends: [0003](0003-rescue-steps.md), [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0062](0062-workflow-descriptors.md) and [0108](0108-test-support.md)

## Summary

Two limits, both set by the workflow and measured on the executor's clock. An **attempt timeout**, on a step descriptor, limits how long one attempt may take, from the start of building the step until its run returns; an attempt over the limit fails the step, whatever it returned. A **journey timeout**, on the workflow descriptor, limits how long a journey may run; once it has passed, the journey fails at its next executor checkpoint. Neither interrupts running code, so both mean the same in every language and execution mode.

Part of one set with proposals 0133 (hooks before and after a journey, a step and an attempt), 0134 (pending steps), whose clock it measures time with, and 0130 (cancellation), at whose executor checkpoints it checks its limits. References such as "#134" are to the issues of those proposals; numbered references such as "6.1 point 4" are to the chapters of specification 0.1.

## Motivation

1. **The workflow sets rules on the steps it includes.** A step is reusable, and today only the step decides how long its work may take (4.8 point 3). A workflow that includes it has no way to say that, in this journey, an attempt taking more than five seconds is a failure.
2. **Journeys that wait.** With pending steps (#134), a journey may wait for a long time. Each step's pending budget bounds it, but nothing bounds the journey as a whole.
3. **Steps that set no limit.** A step that never sets a timeout on the calls it makes can make a journey run much longer than its workflow intends.

## Design

### Words

1. **Attempt timeout:** the most time one attempt of a step may take, as below.
2. **Journey timeout:** the most time a journey may run, as below.
3. **Checkpoint**, **executor checkpoint** and **language checkpoint:** as #130 defines them.
4. Chapter 1 gains the first two terms.

### Declarations

1. **An attempt timeout** MAY be set on a step descriptor: a duration greater than 0. It applies to every attempt of that step, and not to its rescue step.
2. **A journey timeout** MAY be set on the workflow descriptor: a duration greater than 0.
3. **Durations are whole milliseconds,** as #134's "The clock" point 4 says for every duration of tier 2: each language accepts its own standard form, such as a duration type or an object naming days, hours, minutes, seconds and milliseconds, and converts it exactly. How each language spells the setting belongs to its tech spec; the specification's name for it is **timeout**, on the step descriptor for the attempt timeout and on the workflow descriptor for the journey timeout.
4. **A timeout of 0 or less, or with a fraction of a millisecond,** is the new violation `invalid timeout`, found when the workflow is built (3.6, 3.7). It is never rounded.
5. **The listing** shows each step's attempt timeout and the workflow's journey timeout.

### The attempt timeout

1. **What it measures.** For each attempt of a step with an attempt timeout, the executor reads its clock right after the attempt's `before attempt` hooks (#133), before the step's inputs, services and constructor are resolved and called, and again when the step's run returns, whatever it returned. Input adapters and service providers are the workflow's own code, but they serve this step, so their time counts against the attempt.
2. **Over the limit:** when the time taken, in whole milliseconds, is greater than the attempt timeout, the attempt **timed out**, whatever it returned: a success, a failure, a skip, a pending (#134) or an abnormal termination. In order:
   1. the outcome fact the attempt reported, since it happened;
   2. the new fact `attempt_timed_out`;
   3. the attempt's `after attempt` hooks (#133), with the new attempt ending `timed out`;
   4. its contributions are not committed: `contributions_discarded`;
   5. no outcome hook other than `on step failure`, no retry, no rescue and no deferral: `step_given_up` with the new cause `attempt timed out`, decided by `default`. This amends 7.2 point 1, under which a failed attempt of a step with a rescue step is rescued; 2.7 point 2, since a step decision then follows a success or a skip; and 2.7 and 2.8 point 3, since `contributions_discarded` then follows an attempt that did not skip;
   6. the step's `after step` hooks, with the ending failed;
   7. `on step failure`, then the journey fails by default, as for any step given up. If it returns `FailWorkflow`, the journey fails with that reason, as 6.3 point 4 says, and `step_given_up` still carries `attempt timed out`.
3. **Within the limit:** nothing changes.
4. **What it does not do.** It never interrupts an attempt (4.8 point 1): a step that never returns is not caught by it, and is what the journey timeout and cancellation are for. A step still sets limits on the calls it makes (4.8 point 3).
5. **An abort while the step is built** ends the journey as today; nothing is measured.
6. **The order at a checkpoint.** The attempt timeout is checked only at the executor checkpoint `after the step returns` (#130), after cancellation and the journey timeout: a journey cancelled there is cancelled, and one whose journey timeout has passed there fails for the journey timeout, with no `attempt_timed_out`.

### The journey timeout

1. **What it measures:** the time since `journey_started`, on the executor's clock, in whole milliseconds.
2. **Which time counts is part of an executor's capability.** The local executor counts all time since `journey_started`, its waits for nbf included. An executor that stores a journey and resumes it later, such as a durable executor in a later tier, MUST say whether the time a journey spends stored counts towards its journey timeout, and MAY leave it out. The attempt timeout is not affected, since an attempt never runs across a pause.
3. **Where it is checked:** at every executor checkpoint (#130), after cancellation. A wait of the executor, such as a wait for a pending step's nbf (#134), ends at the moment the journey timeout passes, so the checkpoint `while waiting` sees it at once. It is never checked at a language checkpoint, since nothing is delivered inside custom code for it. Once found, it is not checked again: the checkpoints of what follows do not see it.
4. **When it has passed at a checkpoint,** the journey fails, in order:
   1. the new fact `journey_timed_out`;
   2. the step in progress, if any: a step whose turn began (#133) and whose status is `NotExecuted`, `Executing`, `MustRetry` or `Rescuing`. Before it is given up:
      1. an attempt that has just returned, found at `after the step returns`, keeps its outcome fact, and its contributions are not committed;
      2. an attempt that started and whose step's run was not called, found at `before building` or `before hook`, has no outcome fact, as for a cancellation there (#130), which amends 2.7 point 1;
      3. for either attempt, its `after attempt` hooks run as point 4 of #133's "The hooks" says, with the ending `timed out`;
      4. a rescue step that had started becomes `Abandoned`, as for a cancellation (#130), and the step it rescues is the one given up.

      Then `step_given_up`, with the new cause `journey timed out`, decided by `default`, naming the step's last attempt, or no attempt when it had none, so that a `NotExecuted` step goes to `Failed` as when `before step` fails (#133). Then its `after step` hooks, with the ending failed. No outcome hook runs, `on step failure` included: the journey timeout is the workflow's decision about the whole journey, and the last attempt's policy instances may already be gone (6.1 point 5);
   3. every `Pending` or `Waiting` step is abandoned, as #134's "Ending while steps are pending" says;
   4. the journey's end, as #133 says: `after journey`, `on workflow failure`, then `journey_failed` with the cause `journey timed out`, naming the step it was on, if any, decided by `default`.
5. **No checkpoint remains** once the journey's ending is known, when its `after journey` hooks start (#130): a journey timeout that passes after that changes nothing.

### Test support

1. **The path** (8.3) gains the lines `<step> attempt <n> timed out`, `journey timed out at <step>: <checkpoint>` or `journey timed out: <checkpoint>`, and the causes `attempt timed out` and `journey timed out` for `<step> given up`. As for every line of the path, the recording reporter assembles each one from the fields of its event; no event carries ready-made text. A journey failed by a timeout has neither a reason nor an error, so its `journey failed` line writes the cause instead: `journey failed at <step>: attempt timed out`, `journey failed at <step>: journey timed out`, or `journey failed: journey timed out` when no step is named (#133 gives the form with no step). This amends the `journey failed` line of 8.3 point 3, which does not repeat the cause: the cause is written only when there is neither a reason nor an error. 8.3 point 2 is amended, since `attempt_timed_out` and `journey_timed_out` are neither outcomes nor decisions.
2. **Scripted runs** (8.5) gain `taking` a duration, after any ending: the executor's clock moves on by that duration during the attempt, as the runner's clock does in conformance (#134, "The clock").

### Relation to other rules

1. **4.8** stays true: the executor never interrupts custom code, and a step still handles its own time limits; the attempt timeout is the workflow's rule on how long an attempt may take, applied once it returns. 4.8 point 3 gains that a workflow may also set one.
2. **#134's pending deadline** stays: it bounds how long one step may keep deferring, while the journey timeout bounds the whole journey.
3. **#130:** cancellation is checked first at every checkpoint, because the caller asked to stop.

### Implementation ideas

The executor keeps the journey's deadline beside the pending steps' nbfs, so one wait on the clock covers both. Nothing here needs a timer that interrupts code. How each language spells the setting is in its implementation issue: [itinera-rs#199](https://github.com/itinera-dev/itinera-rs/issues/199), [itinera-ts#95](https://github.com/itinera-dev/itinera-ts/issues/95) and [itinera-j#99](https://github.com/itinera-dev/itinera-j/issues/99) (Kotlin and Java).

## Events

| Event | Kind | When | Carries |
|---|---|---|---|
| `attempt_timed_out` | fact | an attempt took more than its attempt timeout | step, attempt, the attempt timeout and the time taken, in milliseconds |
| `journey_timed_out` | fact | the journey timeout had passed at an executor checkpoint | the journey timeout in milliseconds, the checkpoint, the step in progress and its attempt, if any |

1. **New causes** on `step_given_up`: `attempt timed out` and `journey timed out`; on `journey_failed`: `journey timed out`, and `attempt timed out` through the step given up.
2. **New attempt ending** `timed out`, beside #133's.
3. **The result** (5.8): the table gains rows for both causes, with no reason and no error.
4. **Order** (2.8): `attempt_timed_out` comes right after the outcome fact; `journey_timed_out` is the first event after the checkpoint where the journey timeout was found.
5. **Events with no step:** `journey_timed_out` at a checkpoint with no step in progress is about the journey, as `journey_failed` is (2.5 point 4).

## Conformance

Cases, tagged with this proposal and `@tier-2`, with the runner driving the executor's clock and scripted attempts `taking` a duration. They check:

1. an attempt within its limit, and one just over it;
2. an attempt that succeeds over its limit: its outcome fact, `attempt_timed_out`, `after attempt` with the ending `timed out`, contributions discarded, `step_given_up` with `attempt timed out`, `after step`, `on step failure`, and the journey failed;
3. an attempt over its limit with retry budget left, a rescue step or a pending outcome: no retry, rescue or deferral;
4. time spent in input adapters and service providers counted against the attempt;
5. a journey timeout passing during a step, between attempts, and while waiting for a pending step's nbf, with `journey_timed_out` naming the checkpoint once, the step given up with no outcome hook, pending steps abandoned, and the journey failed with `journey timed out`; and a journey timeout found while a rescue step runs;
6. the order at `after the step returns`: cancellation first, then the journey timeout, then the attempt timeout;
7. a journey timeout passing once `after journey` started: no effect;
8. the violation `invalid timeout`, and the listing;
9. the new path lines and the result's rows.

## Alternatives considered

1. **An attempt timeout that interrupts the step:** Kotlin and Java could stop it, async Rust could drop it, and synchronous Rust and TypeScript could not, so the same journey would behave differently by language.
2. **A timed out attempt retried like an abnormal termination:** a slow attempt that succeeded may already have had its effects, and trying it again would repeat them; the workflow asked for a limit, so passing it fails the step.
3. **A step timeout across all attempts:** the workflow's need is to limit one attempt; the journey timeout bounds the rest.
4. **Building not counted:** input adapters and service providers serve the step, so their time is part of what the attempt costs.
5. **A journey timeout given to `run`:** a caller who needs a limit of its own can cancel (#130); the timeout belongs to the workflow's definition.

## Open questions

None.
