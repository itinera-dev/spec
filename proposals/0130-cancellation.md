# 0130: Cancelling a journey, and a hook when a journey is dropped

- Issue: [#130](https://github.com/itinera-dev/spec/issues/130)
- Tier: 2
- Amends: [0003](0003-rescue-steps.md), [0008](0008-steps.md), [0009](0009-running-a-workflow.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0024](0024-abnormal-termination-retriable.md), [0054](0054-how-custom-code-fails.md), [0085](0085-aborted-result-has-no-data.md) and [0108](0108-test-support.md)

## Summary

Define what happens when the caller of `run` asks a journey to stop. Two cases are kept apart. **Cancellation** is a request: the caller triggers a cancellation token given to `run`, in every language and both execution modes, and the journey stops at the next checkpoint. Cleanup runs (the after hooks of #133 that have not run yet, with the ending `cancelled`), a final event `journey_cancelled` is delivered, and `run` ends as the language's cancellation does. A **dropped journey** is a running journey its caller dropped, where the language allows it, such as a Rust future dropped mid-await: nothing more of it can run, except one new synchronous hook, `on journey dropped`, where the language can call it.

Part of one set with proposals 0133 (hooks before and after a journey, a step and an attempt) and 0134 (pending steps), whose terms it uses, and 0135 (timeouts), which checks its limits at this proposal's executor checkpoints. References such as "#134" are to the issues of those proposals; numbered references such as "6.1 point 4" are to the chapters of specification 0.1.

## Motivation

1. **Nothing is defined today.** The specification says the executor never interrupts custom code (chapter 4, 4.8), but not what happens when the caller stops `run`. The implementations differ: in Kotlin a cancelled coroutine stops when custom code next suspends, with no further event; in Rust a dropped future stops at the `.await` it was paused on, with no event and no result; in TypeScript a promise cannot be cancelled, so the journey runs on in the background.
2. **What tier 1 requires is wrong for a caller who asked to stop.** On the JVM, an interrupt reaches a step's blocking call as a recoverable exception, which is an abnormal termination (5.10 point 1). 5.9 point 4 and 5.10 point 2 then require the journey to go on by its usual rules, so a retry or a rescue runs after the caller asked the journey to stop.
3. **Cleanup and audit.** Reporters see a cancelled journey stop without an end, and nothing opened for a step, such as a transaction through a role (#133), is closed. An application cannot learn that a journey it started was cancelled, for example to mark an order for manual follow-up.

## Design

### Words

1. **Checkpoint:** a point at which a journey checks whether it must stop: an executor checkpoint, or a language checkpoint ("Cancellation" point 3). The timeouts of #135 are checked at the executor checkpoints too.
2. **Cancellation:** the caller's request that a running journey stop, made by triggering the journey's cancellation token. A **cancelled journey** is one that stopped for it.
3. **Dropped journey:** a running journey that its caller made impossible to continue, such as a dropped Rust future. Nothing of it can run any more, so it is not cancelled: it simply stops.
4. Chapter 1 gains these terms.

### Cancellation

1. **A token on `run`.** Every executor's `run` MUST accept an optional cancellation token, in both execution modes. A language's own cancellation mechanisms count as triggering it: for example TypeScript's `AbortSignal` is the token, and a cancelled Kotlin coroutine or an interrupted JVM thread counts as a triggered token. How each language spells the token belongs to its tech spec. A caller who only stops waiting for the result cancels nothing; the journey runs to its end.
2. **Before the journey.** A token triggered when `run` is called, or before `journey_started`, means there is no journey: no event and no hook, and `run` ends as cancelled (point 10).
3. **Checkpoints.** When the token is triggered after `journey_started` and before the journey's last event, the journey MUST stop at the next checkpoint. There are two kinds:
   - **executor checkpoints,** where the executor checks: a closed set, each with the name `journey_cancelled` carries, listed below;
   - **language checkpoints,** wherever the language's runtime delivers cancellation inside custom code, as point 4 says, at moments the runtime decides. `journey_cancelled` names them `language checkpoint`.

   The executor checkpoints are:
   1. `before step`: when the scan reaches a step, before its turn begins (#133), so before its step policies are built for the step and before its `before step` hooks. A step cancelled here has not had its turn: it stays `NotExecuted`, and none of its hooks run;
   2. `before attempt`: before a step's `attempt_started`, a retry's and a rescued step's next attempt included, but not a rescue step's;
   3. `before building`: before a step policy is built, for the step (#133) or for an attempt, and before an input adapter, a service provider or a step's constructor is called for an attempt;
   4. `after the step returns`: once a step or rescue step returns, after its outcome fact and before its `after attempt` hooks;
   5. `before hook`: before any hook other than the after hooks of #133, except where the point is already `before step`;
   6. `before rescue`: before a rescue step's `attempt_started`;
   7. `while waiting`: during a wait, such as a wait for a pending step's nbf (#134), where it stops at once.

   Each point has one name. For one attempt they come in this order: `before attempt`; `before building`, for its policies; `before hook`, before each `before attempt` hook; `before building`, for its inputs, services and constructor; `after the step returns`; then `before hook`, before each outcome hook.

   The executor MUST NOT interrupt custom code to stop the journey (4.8). No checkpoint remains once the journey's ending is known, when its `after journey` hooks start (#133): a token triggered after that changes nothing, and `run` returns the journey's result.
4. **Language checkpoints: cancellation delivered inside custom code.** While a checkpoint remains (point 3), where a language delivers cancellation to running custom code as a failure, such as `CancellationException` at a Kotlin suspension point or `InterruptedException` from a blocking call on the JVM, directly or as the cause of another failure, that failure is the cancellation, whatever the custom code is: a step or rescue step, a policy, an input adapter, a service provider, a constructor, a value conversion, a hook, a role operation, a reporter or a dispatcher. It is not an abnormal termination, an abort, `hook failed` or `reporter failed`. The call ends, the journey is cancelled at that language checkpoint, and nothing is retried or rescued. It counts as the cancellation only when this journey's token was triggered before the failure; otherwise it is an ordinary failure, with the outcome 5.10 point 2 gives it. Once no checkpoint remains, such a failure is an ordinary failure too, with the outcome 5.10 point 2 and #133 give it. When it is delivered inside a reporter during the journey's normal flow, that reporter's call ends, the event is still delivered to the reporters after it, and that reporter goes on receiving the cleanup events and `journey_cancelled`, since it did not fail. 5.10 point 2's table gains this outcome for every row during a journey, and its point 3 says so.
5. **What runs, in order:**
   1. **The attempt that was running,** if any, ends:
      1. ended by the cancellation delivered inside it: it has no outcome fact and no step decision;
      2. cancelled at a checkpoint after its `attempt_started` and before its step's run was called, at `before building` or `before hook`: it has no outcome fact and no step decision;
      3. returned, with the cancellation honoured at `after the step returns`: its outcome fact is emitted, since it happened, but nothing follows from it: its contributions are not committed, and no outcome hook, step decision, retry or rescue follows. A token triggered later, during the attempt's `after attempt` hooks, is honoured at the next checkpoint: those hooks were given the attempt's own ending, and its contributions are committed or discarded as usual.

      This amends 2.7 points 1 and 2.
   2. **`after attempt`** (#133) for that attempt, with the ending `cancelled`, as point 4 of #133's "The hooks" says. For an attempt that returned, it MAY request the outcome the attempt reported, its reason, its error and its contributions, as after any other ending.
   3. **`after step`** (#133) for the steps point 5 of #133's "The hooks" names: the step that was running, and every `Pending` or `Waiting` step (#134), with the ending `cancelled`. Those steps become `Abandoned`, as #133's "Statuses and endings" table says, from whatever status they had: `NotExecuted` for a step whose turn began before its first attempt started, `Executing`, `MustRetry`, `Rescuing`, `Pending` or `Waiting`. A rescue step running when the journey is cancelled becomes `Abandoned` too; it has no hooks, and `journey_cancelled` names it, with the step it rescues. This amends 5.4, 5.5 and its diagram, 5.7 and 7.5.
   4. **`after journey`** (#133), with the ending `cancelled`.
   5. **The fact `journey_cancelled`,** the last event of the journey.
6. **No after hook runs twice.** Cleanup runs only the after hooks that have not run yet in the journey. A step that is already final, because its `after step` hooks returned, is left as it is: a cancellation honoured before `on step success` or `on step failure` runs no `after step` again. An after hook inside which the cancellation is delivered during the journey's normal flow was called, and does not run again; the after hooks of its level not yet called run in cleanup, with the ending `cancelled`.
7. **What does not run:** any further step or attempt; the outcome hooks (`on step success`, `on step failure`, `on step retry`, `on step abnormal termination`, `on step pending`); rescue steps; before hooks of #133 not yet called; `on workflow success` and `on workflow failure`.
8. **Cleanup runs undisturbed.** The after hooks MUST run as if no cancellation were pending, so that their blocking or suspending calls work, such as a rollback over the network; the language's own cancellation state is restored when `run` ends. A second cancellation during cleanup changes nothing.
9. **Abort and cancellation.** Whichever comes first stands. An abort that happens before the cancellation is honoured at a checkpoint ends the journey as aborted, the token is then ignored, and `run` returns the aborted result. Once a cancellation is honoured, nothing can abort the journey, as "Failures during cleanup" says.
10. **What `run` gives.** The caller asked to stop, so there is no journey result: `run` ends in the language's own way for a cancelled call, for example by rethrowing `CancellationException` in Kotlin or rejecting with the signal's reason in TypeScript. That is not an exception "for anything that happens inside the journey"; it is the caller's own request coming back. This amends 5.9 point 4.
11. **The event** `journey_cancelled`: a fact, carrying the checkpoint where the cancellation was honoured, the step that was running or waited for, if any, its attempt, if any, and the steps that became `Abandoned`.

### Failures during cleanup

1. **After hooks.** A failing after hook is recorded with `after_hook_failed` (#133), the other after hooks still run, and nothing it does can abort the journey or change how it ends. A request that would abort, something that is not a value or a reserved metadata key stops that hook only, recorded the same way.
2. **Reporters and dispatchers.** Unlike after an abort, the events and global metadata of cleanup are delivered, since cancellation is cooperative. A reporter or a dispatcher that fails during cleanup does not abort: the reporter that failed receives nothing more, and delivery goes on to the others, `journey_cancelled` included. One that fails while `journey_cancelled` is delivered is ignored, as 2.3 point 4 says for `journey_aborted`, and delivery continues to the remaining reporters.
3. **Unrecoverable failures** (5.10 point 5) stay outside the model: nothing runs after one.

### The dropped journey

1. **What it is.** Where a language lets a caller drop a running journey so that nothing more of it can run, the journey stops where it is, as with an unrecoverable failure (5.10 point 5): no further event is delivered, and none of #133's hooks run.
2. **`on journey dropped`,** a new workflow hook, is the one thing that runs, where the language can call it, for example from a guard's `Drop` in Rust:
   1. it is defined at most once per workflow, like today's workflow hooks (3.3 point 4, `hook defined twice`);
   2. it runs once, when and only when the journey is dropped after `journey_started` and before its last event, during cleanup included;
   3. it is called synchronously, in every execution mode; it SHOULD NOT block, and the executor does not wait for anything it starts, such as a task of its own or a message on a channel;
   4. it MAY request the journey ID and roles (6.8). It MUST be synchronous, and so MUST every role operation it calls: an asynchronous one, in either execution mode, is the new violation `asynchronous drop hook`, found when the workflow is built (3.6);
   5. it decides nothing: it is void, and what it returns is never read;
   6. a failure in it is ignored, or passed on in the language's own way.
3. A language that cannot drop a running journey never calls it, and its runner excludes the cases as impossible, with its proof.
4. The 6.3 table and 6.4 gain it, and 5.10 point 2 gains its row.

### Amendments

1. **Chapter 2:** 2.4 point 3 gains that `journey_cancelled` is the last event of a cancelled journey; 2.7 gains the fact `journey_cancelled`, and its points 1 and 2 the exceptions of "What runs" point 1; 2.8 point 5 gains `journey_cancelled` beside `journey_aborted`; 2.3 points 1 to 3 gain "Failures during cleanup" point 2.
2. **Chapters 4 and 5:** 4.4 point 1 and 5.10 point 1, since a cancellation delivered inside a step is not an abnormal termination (#8, #24); 5.9 point 4 (what `run` gives); 5.10 points 2 and 3 (cancellation inside custom code, and `on journey dropped`); 5.8 point 1, since a cancelled journey has no result, in line with an aborted result carrying no data bag (0085).
3. **Chapter 6:** the new workflow hook `on journey dropped`.
4. **Chapter 8:** 8.3 gains the path line `journey cancelled at <step>: <checkpoint>`, or `journey cancelled: <checkpoint>` when no step concerns it; 8.5 point 5.4 gains the ending `will wait for cancellation`, and 8.5 point 9 that a scripted run whose journey is cancelled gives its record, and in place of the result says that the journey was cancelled. Test support MAY offer applications the same way to cancel: a recording reporter that triggers a token when a named event is delivered, before it returns.

### Implementation ideas

The language-specific points, such as each language's token, its language checkpoints and how it drops a journey, are in the implementation issues linked under "Conformance".

## Events

1. **New fact `journey_cancelled`,** the last event of a cancelled journey: the checkpoint where the cancellation was honoured, the step that was running or waited for, if any, its attempt, if any, and the steps that became `Abandoned` ("Cancellation" point 11).
2. **New workflow hook `on journey dropped`,** whose calls emit nothing, since a dropped journey delivers no further event ("The dropped journey").
3. The events of cleanup, the `hook_called` and `after_hook_failed` of the after hooks of proposal 0133, are delivered, unlike after an abort ("Failures during cleanup" point 2).


## Conformance

**How a case cancels.** A case never cancels after a delay or from a thread of its own, so its outcome never depends on scheduling. The runner plays the caller, and the token is triggered in reaction to an event of the journey, before the journey can reach its next checkpoint:

1. **`Given the caller cancels the journey on "<event>"`,** optionally `of "<step>"`: when the named event is delivered, the token MUST be triggered before the recording reporter returns from it. A runner MAY do this through a queue, with a listener that triggers the token and acknowledges while the reporter waits; what matters is that the reporter returns only after the token is triggered. Since a dispatcher only calls reporters, and a reporter that has not returned holds the journey (2.2 point 7), the journey cannot reach its next checkpoint before that.
2. **`Given the caller cancelled the journey before it runs`:** the token is triggered before `run` is called.
3. **`When the caller drops the journey on "<event>"`:** where a language can drop a running journey, the runner drops it once the named event is delivered.
4. **`Then the run ended as cancelled`:** `run` ended in the language's own way for a cancelled call (point 10 of "Cancellation").
5. **A new scripted ending, `will wait for cancellation`** (8.5 point 5.4): the attempt waits in the language's cancellation-aware way, and ends as that cancellation ends it. A step that waits after emitting an event of its own, through its step reporter, lets a case cancel on that event and observe a language checkpoint. In a language that has no language checkpoints, the attempt waits until the journey's next executor checkpoint, or until it is dropped.
6. **Each executor checkpoint is reached** from the event just before it: `after the step returns` from the attempt's outcome fact, `before hook`, `before rescue` and `before attempt` from the step decision before them, and `while waiting` from `journey_waiting`, which is delivered before the executor waits.

How each runner triggers the token, drops a journey and waits for cancellation belongs to its implementation issue: [itinera-rs#196](https://github.com/itinera-dev/itinera-rs/issues/196), [itinera-ts#94](https://github.com/itinera-dev/itinera-ts/issues/94) and [itinera-j#98](https://github.com/itinera-dev/itinera-j/issues/98) (Kotlin and Java).

Cases, tagged with this proposal and `@tier-2`, check:

1. a cancellation during a step that returns: its outcome fact kept, its contributions not committed, no outcome hook, step decision, retry or rescue, `after attempt` given the ending `cancelled` and the reported outcome, the after hooks nested, then `journey_cancelled` last;
2. a cancellation delivered inside a step, an input adapter, a service provider and a hook: no outcome fact for an attempt it ends, no abort, and the checkpoint `language checkpoint`. These cases are tagged `@language-checkpoint`; a language that has no language checkpoints excludes them, saying so in its tech spec;
3. a cancellation during a wait for nbf: immediate, with the checkpoint `while waiting`, `Pending` and `Waiting` steps abandoned, and their `after step` hooks run;
4. a cancellation at each other executor checkpoint: what comes after it does not run, and `journey_cancelled` names the checkpoint;
5. a cancellation before `on step success`: no `after step` runs twice;
6. a token triggered before the journey starts: no event, and `run` ends as cancelled;
7. a failing after hook and a failing reporter during cleanup: the others still run, no abort, and `journey_cancelled` delivered;
8. `on journey dropped` running once for a dropped journey, and never on success, failure, abort or cancellation;
9. the path line `journey cancelled`, and a scripted run that is cancelled;
10. a token triggered once `after journey` started: `run` returns the result;
11. an abort before the cancellation is honoured: the journey ends aborted;
12. a failure of the type a language uses for cancellation, with the token never triggered: an ordinary failure.

What a runner cannot trigger, such as a dropped journey in a language that cannot drop one, is excluded as impossible, with its proof.

## Alternatives considered

1. **Keep today's behaviour undefined:** the languages disagree, and a JVM interrupt is retried.
2. **Only a final event, without cleanup:** leaves what a before hook opened open.
3. **Running `on workflow failure` on cancellation:** blurs a business outcome with the caller leaving.
4. **One synchronous, non-blocking cancellation hook for every case,** this issue's first design: forced by Rust's drop, it is too little for cooperative cancellation, where cleanup can wait, use roles and read data.
5. **Synchronous cancellation as a capability:** a token works in every language and both modes, so no capability is needed.
6. **A `step_abandoned` decision for each step a cancellation abandons:** nobody decides it, since the caller's request is a fact; `journey_cancelled` names those steps instead.

## Open questions

None.
