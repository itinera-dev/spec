# 8. Test support

Specification 0.1. Written from proposal [0108](../proposals/0108-test-support.md), as amended by [0119](../proposals/0119-attempt-mistakes.md).

Test support is a capability (chapter 1, 1.8) that lets an application test its workflows: a recording reporter and the path it reads, fake providers, scripted runs and recorders. It never changes what a journey means; it only lets a test replace custom code, call it outside a journey, or read what a journey did. A language that does not claim it need offer none of this.

## 8.1 The capability

1. **Test support** is a capability (chapter 1, 1.8) a language claims for its implementation. A language that claims it MUST offer all five parts, 8.2 to 8.6, and pass the cases tagged `@capability-testing`.
2. Test support SHOULD be available only where the application asks for it, such as a build used for tests, where the language allows it.
3. **It does not count towards a tier.** A language claims tier 1 with or without test support; its cases run, and proposal 0108 is listed in a language's manifest, only when the language claims `testing`.
4. Nothing in test support changes how a journey runs: hooks, policies, decisions, events, aborts and the result are the executor's, exactly as in a journey without it. Where a part replaces custom code (a fake provider, a scripted attempt), everything around that code happens as for the code it replaces.
5. **A mistake in a test** (8.4 point 5, 8.5 points 7 to 9), such as a script that leaves out an attempt or a fake named after a provider the workflow does not have, MUST stop the test where it is made, as the language reports a misuse in a declaration: a panic in Rust, an exception elsewhere. A mistake in a fake provider or a script stops it before the journey starts; a mistake in reading a scripted run's record stops it where the record is read, after the journey, which stays as it ended. It is not a refusal and no event is emitted. Its message MUST name each problem; this specification says what is named, not the wording.

## 8.2 The recording reporter

1. A **recording reporter** is a reporter that keeps every event it receives, in order, and the latest value of each entry of global metadata it receives.
2. A test MUST be able to add it to an instance and read what it kept once the journey is over: its `events`, its `global metadata`, and its `path`.

## 8.3 The path

1. A journey's **path** is a list of lines, one per event of the journey's flow, in the order of the events, read from a recording reporter.
2. **Only the flow is listed:** each attempt's outcome, each step decision, and how the journey ended. Inputs, services, hooks, contributions, events steps and hooks emit, global metadata, timestamps, sequence numbers and IDs MUST NOT appear, so a path changes only when the flow does.
3. **The lines** MUST read exactly as follows, where `<step>` is the step's name, a rescue step's included, `<n>` the attempt's number, and a reason is written by its code:

   | Event | Line |
   |---|---|
   | `step_succeeded` | `<step> attempt <n> succeeded` |
   | `step_failed` | `<step> attempt <n> failed: <reason>` |
   | `step_skipped` | `<step> attempt <n> skipped`, followed by `: <reason>` when it has one |
   | `step_abnormal_termination` | `<step> attempt <n> terminated abnormally: <error message>` |
   | `step_retrying` | `<step> retrying` |
   | `step_given_up` | `<step> given up: <cause>`, where the cause is `failure`, `abnormal termination`, `retries exhausted`, or `FailWorkflow from <policy>` |
   | `step_rescuing` | `<step> rescuing with <rescue step>` |
   | `step_rescued` | `<step> rescued by <rescue step>` |
   | `journey_succeeded` | `journey succeeded`, or `journey succeeded, decided by <policy>` when a hook returned `FinishWorkflow` |
   | `journey_failed` | `journey failed at <step>: <reason>`, with the reason the journey failed for, the last attempt's when the retries were exhausted, or the error message when there is no reason, as in `journey failed at charge: gateway down`; the cause is not repeated, since the `given up` line before it carries it |
   | `journey_aborted` | `journey aborted at <step>: <abort reason>` when the abort concerns a step, as in `journey aborted at charge: step could not be built`, and `journey aborted: <abort reason>` otherwise, as in `journey aborted: reporter failed` |

4. A refused journey has no events, so its path is empty.

For example, a journey whose `charge` is declined once, retried and then succeeds, followed by `ship`:

1. `charge attempt 1 failed: declined`
2. `charge retrying`
3. `charge attempt 2 succeeded`
4. `ship attempt 1 succeeded`
5. `journey succeeded`

## 8.4 Fake providers

1. A test MAY replace one **service provider** of an instance, named by the provider's name, with a **fake provider**: a function of the test's own.
2. The fake replaces only the provider's function. The provider's attachment, scope and declared requests stay as declared: the requests are resolved first, as for the provider, so missing data aborts the journey as it would otherwise, and the fake is called wherever and as often as the provider would be, once per journey or once per attempt.
3. The fake returns what a provider returns (chapter 4): an instance, optionally with data, or a failure. Everything that follows is as for the provider: `service_provided` names the fake's own implementation and carries its data, and a fake that fails aborts the journey with `step could not be built`, as a provider does.
4. The workflow's declaration, its admission and its listing are unchanged.
5. A fake named after a provider the workflow does not have, or making another service than the provider makes, is a mistake in the test, reported before the journey starts. Each fake replaces one provider: a workflow with two providers of one service gets one fake for each it replaces.

## 8.5 Scripted runs

1. A test MAY run an instance **scripted**: with a script that says, for the steps it names, what their attempts do instead of running the steps.
2. **Every step is still built for real**, its inputs, its services and its constructor included, so a missing input, a failing provider or a failing constructor aborts the journey as it would otherwise. Only the call to the step's `run` is replaced. A step the script does not name runs for real.
3. **Naming steps.** The script names steps by their names, with `step` for the first and `and step` for each next one, and rescue steps by their own names. A rescue step has one attempt.
4. **Choosing attempts.** After a step is named, the script says which attempts it describes: `on attempt` with a number, attempt 1 when nothing is said; `on every attempt`; or `on remaining attempts`, every attempt not yet described.
5. **What an attempt does,** in this order:
   1. `will contribute` a value under a key, committed only if the attempt succeeds, as a step's contributions are;
   2. `will emit` an event at a level with a message, or `will emit with` data too, as the step's reporter would;
   3. `will set global metadata` an entry to a value, as the step's reporter would;
   4. then an ending: `will succeed`, `will fail` with a reason, `will skip`, `will skip because` of a reason, or `will terminate abnormally` with an error message. Or, instead of all of these, `will run for real`: the attempt runs the real step.
6. **The ending is the step's outcome.** A scripted ending has the same consequences as the outcome a real step reports, or, for `will terminate abnormally`, as an error escaping its `run`.
7. **Checked before the journey.** Before the journey starts, the run MUST stop as a mistake in the test, listing everything the script leaves out, when:
   1. a scripted step does not describe every attempt it can have under its retry budget, its `retry on abnormal termination` setting and its rescue step, given how its earlier attempts end, an attempt scripted to run for real being one that may end in any way; for each such step, the first attempt it leaves out is named, since the attempts after it depend on how that one ends. How the script says the rescue step ends is part of how the step's attempt 1 ends: after that attempt fails or ends in an abnormal termination, the run after the rescue is an attempt the step can have only when its rescue step may succeed, because the script says it succeeds or runs for real, or does not script it. When the script says the rescue step fails or ends in an abnormal termination, the journey fails there (chapter 7, 7.4 point 3), and no further attempt of the step is required; a script that describes one anyway is not a mistake, since that attempt never runs, as an attempt described after one that succeeds never runs;
   2. a scripted step can fail on attempt 1 and its rescue step is not scripted;
   3. the test asked, with `ensure all scripted`, that every step be scripted, and a step is not.
8. **Mistakes in writing the script** MUST be reported where they are written, before the journey, as mistakes in the test: a name the workflow does not have, a step named twice, a gap in the attempts, an attempt chosen twice, an attempt the step can never have, a rescue step that skips, and a contribution, emit or entry of global metadata for a step that declares no contributor or reporter, since the real step could not do it. Choosing attempts is checked as follows:
   1. **An attempt chosen twice** is `on attempt` with the number of an attempt the script already described for the step, or with any number once `on every attempt` or `on remaining attempts` was chosen for it; `on every attempt` once any attempt of the step is described; or `on remaining attempts` once it, or `on every attempt`, was chosen for the step. It names the step and the first attempt the choice describes again: the number given to `on attempt`, attempt 1 for `on every attempt`, and, for `on remaining attempts`, the attempt after the last one described by number, or attempt 1 when none was.
   2. **An attempt the step can never have** is `on attempt` with a number that is not a whole number from 1, such as 0, -1 or 1.5, or with a number greater than the most attempts the step can have (chapter 5, 5.6 point 1): its retry budget plus one, and one more for a step with a rescue step, so a rescue step, which has no retry budget, can have one. So is `on remaining attempts` once the attempts described by number reach that most. It names the step and the attempt: the number given to `on attempt`, and, for `on remaining attempts`, the attempt after the last one described by number. A number that a language's type for an attempt's number cannot hold, such as a fraction where that type holds whole numbers, is impossible to express, and such a language need not check it; it proves this and lists the scenarios it excludes as for the rules of chapter 3, 3.8, points 2 and 3.
   3. **An attempt the step can never have is named first.** When one choice of attempts is both an attempt the step can never have and an attempt chosen twice or a gap in the attempts, the test MUST be stopped naming only the attempt the step can never have: with nothing yet described for a step with a retry budget of 1, `on attempt 5` is an attempt the step can never have, not a gap. A choice is never both an attempt chosen twice and a gap, since the attempts before one already described are described too.
9. **The result.** `run` runs the journey on the local executor of the workflow's execution mode, `sync` or `async`, and returns a **scripted run**: the journey's result and a **record** of every step and rescue step of the workflow. A step's record, read with `step` by name, holds its `attempts`, in order, each attempt handed to the construction of its step, scripted or real; `attempt` reads one by number. An attempt that ended before its step's constructor was called, because an input or a service could not be resolved, is not in it; one whose constructor failed is. Each attempt reads with `input` the values its step was built with and, for a rescue step, the contributions it requested from the failed attempt. Reading with `step` a name that is neither a step nor a rescue step of the workflow is a mistake in the test, an unknown step, naming that name (8.1 point 5). Reading with `attempt` the number of an attempt the record does not hold, such as one its step never reached, or 0, MUST read nothing, as the language says a value is absent, and is not a mistake. A number that the language's type for an attempt's number cannot hold is impossible to express here too, as in point 8.2.
10. A scripted ending MUST also be available to a test tool that writes real steps, as an **ending** that gives the outcome it stands for, so such a tool ends its steps the same way.

## 8.6 Recorders

1. Each handle that the executor lends custom code has a recorder that lends it outside any journey, so a test calls a step's `run`, a hook or an input adapter itself:
   1. **contributions** lends a contributor and keeps what is contributed, read with `read` by key, and `keys`;
   2. **step event reporter** lends a step's reporter, and **hook event reporter** a hook's means of emitting events; each keeps every event emitted, with its level, message and data, and the global metadata set;
   3. a **data bag** built from values, with `with`, lends its read access with `access`.
2. **A lent handle MUST behave as in a journey.** Made `failing`, a reporter recorder interrupts every call and keeps nothing; setting the reserved entry `journey_id` interrupts that call and every later one.
3. What belongs to the journey, such as committing contributions only on success, is not a recorder's: tests that need it run a journey.

## 8.7 Suggested names (not normative)

This section is a suggestion, not part of the specification, as chapter 5, 5.13 is. The operations of 8.2 to 8.6 are named in words, as the specification names its other concepts, such as `on step retry`. A language usually spells them in its own casing, for example `will_contribute` in Rust and `willContribute` in TypeScript and Kotlin, and its tech spec says why when it keeps a different name, such as one that clashes with a reserved word. A language with optional or overloaded arguments may merge an operation with its `with` or `because` form, such as `will emit` and `will emit with`. The names are:

| Part | Names |
|---|---|
| Recording reporter | `recording reporter`, `events`, `global metadata`, `path` |
| Fake providers | `fake provider` |
| Scripted runs | `scripted`, `step`, `and step`, `on attempt`, `on every attempt`, `on remaining attempts`, `will contribute`, `will emit`, `will emit with`, `will set global metadata`, `will succeed`, `will fail`, `will skip`, `will skip because`, `will terminate abnormally`, `will run for real`, `ensure all scripted`, `run`, `scripted run`, `attempts`, `attempt`, `input`, `ending`, `outcome` |
| Recorders | `contributions`, `contributor`, `read`, `keys`, `step event reporter`, `hook event reporter`, `reporter`, `failing`, `with`, `access` |

## 8.8 Checked by

In the conformance repository: [`0108-test-support/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0108-test-support), tagged `@capability-testing`, which only a language claiming test support runs. Names are not checked, since conformance cannot observe them.
