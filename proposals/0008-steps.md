# 0008: Steps

- Issue: [#8](https://github.com/itinera-dev/spec/issues/8)
- Tier: 1

## Summary

What a step is and how it runs: how it is built, what it declares, the outcomes it reports, what happens to the data it contributes, and how it is identified. Builds on the premises of proposal 0002 (Foundations).

## Motivation

A step is where business rules live. For flow control to stay out of it, the step must report what happened in terms of its own domain, never what should happen next, and everything around it (building it, deciding whether to try again, moving on) must be decided elsewhere. Step authors need a small, precise vocabulary for that: did the step complete, could it not complete, and if not, could trying again help; or did it decide there was nothing to do. Workflow authors and operators need to know exactly what each of those means for the journey and for the data.

## Design

### Building a step

1. **Built on demand, once per attempt.** An implementation MUST build a fresh step for every attempt by calling its constructor. Nothing MUST carry over from one attempt to the next.
2. **Declared needs.** A step's constructor declares what it needs:
   - **inputs**, each identified by a key local to the step, a type, and whether it is **required** or **optional**;
   - **a contributor**, through which it contributes data back;
   - **a step reporter**, through which it emits its own events.
3. **Inputs are resolved outside the step,** through the workflow's input adapters or the data bag, as proposal 0002 defines. The step's own code MUST NOT have access to the adapter, the data bag or the resolver.
4. **A step does not know its own name.** Its name is how the workflow refers to it. An implementation MUST NOT provide a step with its name; a step that needs a label receives it as an ordinary input. A step policy MAY request the name of the step it acts on (proposal 0002).

### Two different failures

5. **Building fails: the journey is aborted.** If a step cannot be built, because an input is missing or of the wrong type, an input adapter fails, the constructor fails, or for any other reason, the journey MUST be aborted. It MUST NOT enter the retry loop.
6. **Running fails: an abnormal termination.** If a built step lets an error, an exception or a panic escape while running, the attempt MUST end with an **abnormal termination**. The error is all the hooks receive. By default it is treated like a retriable failure, since nobody knows whether trying again would help.

### Outcomes

7. **An outcome states what happened, never what to do next.** A step that runs to the end MUST report one of:
   - **Success**: the step completed;
   - **Failure**: the step could not complete;
   - **Skipped**: the step decided, from what it knows of its domain, not to do its work.
8. **A failure is not retriable unless the step says it is.** Every failure carries a **retriable** flag, false by default:
   - **not retriable**: trying again would not help. By default the journey fails;
   - **retriable**: trying again might help. By default the step is tried again while its retry budget lasts.

   The flag is a suggestion from the step, not a decision: whether a retriable failure is tried again depends on the step's retry configuration, the hooks and the executor.
9. **Reasons.** A failure MUST carry a reason: a **code** (a string), an optional **message**, and optional serializable **details**. A skip MAY carry a reason of the same shape.
10. **What hooks can request.** The failure hook MAY request the failure's reason and the cause of the call, which is either `failure` or `retries exhausted`. The hook reacting to an abnormal termination MAY request the error. The retry hook MAY request the cause of the call, either `retriable failure` or `abnormal termination`, and the corresponding reason or error. When the retry budget is exhausted, the failure hook MUST be called with the cause `retries exhausted`.

### Skipped

11. **A skipped step is out of the flow.** No hook MUST be called for a skipped step, and no lifecycle can follow it: the executor moves to the next step. A journey whose steps all ended as succeeded or skipped succeeds.
12. **A skipped step has no effect on data.** Anything it contributed MUST be discarded, and an event MUST record that it was.
13. **A skipped step is final.** Its status is `Skipped`; it MUST NOT run again in the journey, and the journey's result lists it as skipped.
14. **Only the step decides to skip.** Hooks cannot skip steps.

### Contributions

15. **Contributions are dynamic.** The contributor accepts a key and a value. Keys are not declared in advance.
16. **The same key twice in one attempt:** the last value MUST win.
17. **Visible to hooks, committed only on success.** Contributions MUST be visible to the hooks of the attempt that made them, whatever the outcome, and MUST be committed to the data bag only when the attempt succeeds. A failed attempt's contributions, retriable or not, MUST NOT reach later steps or the result. An abnormal termination carries no contributions.
18. **Overwriting the data bag.** A committed contribution MAY replace a key already in the data bag. The last committed value wins, and an event MUST record the overwrite.
19. **Values.** A contribution MAY be any value the language can hold. A value of the wrong type is detected when it is read, which aborts the journey (proposal 0002).

### Time

20. **The executor never interrupts a running step.** It does not cancel futures, stop threads or impose time limits on a step while it runs. A step handles its own time limits, for example a timeout on a call it makes, and reports a failure, retriable or not, when it runs out of time.

### Identity and execution mode

21. **Names.** A step's name is a non-empty, case-sensitive string, unique within its workflow. A workflow with two steps of the same name MUST be refused at admission.
22. **Execution mode.** A step is either synchronous or asynchronous; which executors accept which mode is defined by the local executor proposal.

## Events

This proposal adds these engine events, whose fields and place in the catalogue are defined with the event model:

- a step was skipped, with its reason;
- a skipped step's contributions were discarded;
- a committed contribution overwrote a key in the data bag.

## Conformance

The conformance cases for this proposal must check that:

1. every attempt gets a fresh step instance;
2. a step that cannot be built aborts the journey without entering the retry loop;
3. an error escaping a running step is an abnormal termination, retried by default, and the hook can request the error;
4. a failure that is not retriable fails the journey on its first attempt;
5. a retriable failure is tried again while the retry budget lasts, and the retry hook receives its cause;
6. the failure hook receives the reason the step gave;
7. when retries run out, the failure hook is called with the cause `retries exhausted`;
8. a skipped step is out of the flow: no hook is called, the next step runs, and the event carries its reason;
9. a skipped step's contributions are discarded; a later required input aborts the journey and a later optional input is absent;
10. only a successful attempt's contributions are committed;
11. a failed step's contributions reach its hooks but not the data bag;
12. an abnormal termination carries no contributions;
13. the same key contributed twice in one attempt keeps the last value;
14. overwriting a key in the data bag is allowed and recorded;
15. duplicate step names are refused at admission, and names are case-sensitive.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0008-steps/`.

## Alternatives considered

- **A Transient outcome, or a Retry outcome.** Rejected: both describe what to do next, which is a flow decision. A failure with a retriable flag states what the step knows, and leaves the decision to the policy.
- **Calling the flag "temporary" or "definitive".** Rejected: "temporary" suggests the problem is only about timing; "retriable" says what the flag is for.
- **A failure retriable by default.** Rejected: a forgotten flag would cause retry loops; with "not retriable" as the default, it causes a clear failure.
- **A hook reacting to skipped steps.** Rejected: a step that decides to skip leaves the flow.
- **Treating a step that cannot be built as an abnormal termination of the attempt.** Rejected: a step that cannot be built makes the run unsafe, so the journey is aborted.
- **Timeouts enforced by the executor.** Rejected: Itinera's executor is not a low-level runtime that cancels futures or kills threads. A step owns its time limits and reports a failure when they are exceeded.

## Open questions

- A measured time limit, in a later tier: after a step returns, the executor compares how long it took with an allowed duration and fails the step or the workflow if it was exceeded. It never interrupts the step.
- The Pending outcome, with durable runs.
- A hook that runs before a step and may decide it does not run, in a later tier; it would have to change the rule that hooks cannot skip steps, and record such skips as decided by policy.
- Declaring each step's set of failure codes in advance.
