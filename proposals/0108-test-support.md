# 0108: Test support

- Issue: [#108](https://github.com/itinera-dev/spec/issues/108)
- Tier: 1, as the capability `testing`
- Amended by [#119](https://github.com/itinera-dev/spec/issues/119): an attempt scripted twice, an attempt a step can never have, and reading the record of an unknown step.

## Summary

A capability, **test support**, that lets an application test its workflows with the same tools, the same meaning and the same names in every language. It has five parts:

1. a **recording reporter**, which keeps every event and the global metadata of a journey;
2. a journey's **path**, the flow of a journey as plain sentences, read from a recording reporter;
3. **fake providers**, which replace one service provider of an instance by a test's own function, so real steps run against fake services;
4. **scripted runs**, which run a journey with what some attempts do written in a script instead of running their steps, to check a workflow's flow without its steps' business rules;
5. **recorders**, which lend a test the handles only the executor makes otherwise, so a test calls a step's `run`, a hook or an input adapter directly, outside any journey.

The specification defines what each part does. It also suggests, without requiring them, the names of each part's operations, so a test reads the same in every language; each language spells them in its own casing.

Test support is a capability, not part of a tier: a language may claim tier 1 without it.

## Motivation

Itinera takes flow decisions out of business code, so a workflow's flow can only be understood from what the engine did. Tests need the same view. An application author wants to check three things, each without the others getting in the way:

1. **A step's business rule:** call the step with inputs and services, and read back its outcome, contributions and events, without a one-step workflow around it.
2. **A workflow's flow:** which step runs, retries, is rescued or is given up, and how the journey ends, when attempts end in given ways, without the steps' business rules or the outside world.
3. **The whole journey:** real steps against fake services, compared with an approved path.

The first implementation already offers all of this to its tests. Its rules, such as "a scripted step is still built for real" or "a script that leaves out an attempt the step can have stops before the journey", are observable: a scripted run is a real journey, with real events and a real result. By the rule in PROCESS.md, observable behaviour belongs in the behaviour specification. Left in one language's tech spec, a second language could give the same words another meaning, and the book would teach a different testing story in each language.

The names matter as much as the meaning. A test reads as a sentence, such as "step `charge` will fail `declined`, on attempt 2 will succeed", and the book, the site and agents writing tests in any language should find the same sentence. Conformance cannot observe names, so they stay each language's choice, as the instance interface did in proposal 0062. This proposal suggests them, as chapter 5, 5.13 suggests an instance interface, so every language starts from the same sentence.

## Design

### The capability

1. **Test support** is a capability (chapter 1, 1.8) a language claims for its implementation. A language that claims it MUST offer all five parts below and pass the cases tagged `@capability-testing`.
2. Test support SHOULD be available only where the application asks for it, such as a build used for tests, where the language allows it.
3. **It does not count towards a tier.** A language claims tier 1 with or without test support; its cases run, and this proposal is listed in a language's manifest, only when the language claims `testing`.
4. Nothing in test support changes how a journey runs: hooks, policies, decisions, events, aborts and the result are the executor's, exactly as in a journey without it. Where a part replaces custom code (a fake provider, a scripted attempt), everything around that code happens as for the code it replaces.
5. **A mistake in a test**, such as a script that leaves out an attempt or a fake named after a provider the workflow does not have, MUST stop the test before the journey starts, as the language reports a misuse in a declaration: a panic in Rust, an exception elsewhere. It is not a refusal and no event is emitted. Its message MUST name each problem; this specification says what is named, not the wording.

### Suggested names (not normative)

This section is a suggestion, not part of the specification, as chapter 5, 5.13 is. The operations below are named in words, as the specification names its other concepts, such as `on step retry`. A language usually spells them in its own casing, for example `will_contribute` in Rust and `willContribute` in TypeScript and Kotlin, and its tech spec says why when it keeps a different name, such as one that clashes with a reserved word. A language with optional or overloaded arguments may merge an operation with its `with` or `because` form, such as `will emit` and `will emit with`. The names are gathered in "Names at a glance" below.

### The recording reporter

1. A **recording reporter** is a reporter that keeps every event it receives, in order, and the latest value of each entry of global metadata it receives.
2. A test MUST be able to add it to an instance and read what it kept once the journey is over: its `events`, its `global metadata`, and its `path`.

### The path

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

### Fake providers

1. A test MAY replace one **service provider** of an instance, named by the provider's name, with a **fake provider**: a function of the test's own.
2. The fake replaces only the provider's function. The provider's attachment, scope and declared requests stay as declared: the requests are resolved first, as for the provider, so missing data aborts the journey as it would otherwise, and the fake is called wherever and as often as the provider would be, once per journey or once per attempt.
3. The fake returns what a provider returns (chapter 4): an instance, optionally with data, or a failure. Everything that follows is as for the provider: `service_provided` names the fake's own implementation and carries its data, and a fake that fails aborts the journey with `step could not be built`, as a provider does.
4. The workflow's declaration, its admission and its listing are unchanged.
5. A fake named after a provider the workflow does not have, or making another service than the provider makes, is a mistake in the test, reported before the journey starts. Each fake replaces one provider: a workflow with two providers of one service gets one fake for each it replaces.

### Scripted runs

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
   1. a scripted step does not describe every attempt it can have under its retry budget, its `retry on abnormal termination` setting and its rescue step, given how its earlier attempts end, an attempt scripted to run for real being one that may end in any way; for each such step, the first attempt it leaves out is named, since the attempts after it depend on how that one ends;
   2. a scripted step can fail on attempt 1 and its rescue step is not scripted;
   3. the test asked, with `ensure all scripted`, that every step be scripted, and a step is not.
8. **Mistakes in writing the script** MUST be reported where they are written, before the journey, as mistakes in the test: a name the workflow does not have, a step named twice, a gap in the attempts, a rescue step that skips, and a contribution, emit or entry of global metadata for a step that declares no contributor or reporter, since the real step could not do it.
9. **The result.** `run` runs the journey on the local executor of the workflow's execution mode, `sync` or `async`, and returns a **scripted run**: the journey's result and a **record** of every step and rescue step of the workflow. A step's record, read with `step` by name, holds its `attempts`, in order, each attempt handed to the construction of its step, scripted or real; `attempt` reads one by number. An attempt that ended before its step's constructor was called, because an input or a service could not be resolved, is not in it; one whose constructor failed is. Each attempt reads with `input` the values its step was built with and, for a rescue step, the contributions it requested from the failed attempt.
10. A scripted ending MUST also be available to a test tool that writes real steps, as an **ending** that gives the outcome it stands for, so such a tool ends its steps the same way.

### Recorders

1. Each handle that the executor lends custom code has a recorder that lends it outside any journey, so a test calls a step's `run`, a hook or an input adapter itself:
   1. **contributions** lends a contributor and keeps what is contributed, read with `read` by key, and `keys`;
   2. **step event reporter** lends a step's reporter, and **hook event reporter** a hook's means of emitting events; each keeps every event emitted, with its level, message and data, and the global metadata set;
   3. a **data bag** built from values, with `with`, lends its read access with `access`.
2. **A lent handle MUST behave as in a journey.** Made `failing`, a reporter recorder interrupts every call and keeps nothing; setting the reserved entry `journey_id` interrupts that call and every later one.
3. What belongs to the journey, such as committing contributions only on success, is not a recorder's: tests that need it run a journey.

### Names at a glance (not normative)

| Part | Names |
|---|---|
| Recording reporter | `recording reporter`, `events`, `global metadata`, `path` |
| Fake providers | `fake provider` |
| Scripted runs | `scripted`, `step`, `and step`, `on attempt`, `on every attempt`, `on remaining attempts`, `will contribute`, `will emit`, `will emit with`, `will set global metadata`, `will succeed`, `will fail`, `will skip`, `will skip because`, `will terminate abnormally`, `will run for real`, `ensure all scripted`, `run`, `scripted run`, `attempts`, `attempt`, `input`, `ending`, `outcome` |
| Recorders | `contributions`, `contributor`, `read`, `keys`, `step event reporter`, `hook event reporter`, `reporter`, `failing`, `with`, `access` |

## Events

None added or changed. Test support reads events; it emits only what the parts it replaces would emit.

## Conformance

The cases are tagged `@capability-testing` and run only for a language that claims test support. Its conformance runner runs them through the language's own test support. The cases must check that:

1. a recording reporter keeps every event in order, and the latest value of each entry of global metadata;
2. a journey's path lists exactly the lines above for successes, failures with and without retries, skips with and without a reason, abnormal terminations, rescues, `FailWorkflow`, `FinishWorkflow`, failed journeys and aborted ones, and is empty for a refusal;
3. a fake provider is called where its provider would be, per journey or per attempt as declared; its provider's requests are resolved first; `service_provided` names the fake's implementation and data; a fake that fails aborts with `step could not be built`; and an unknown provider name or another service is reported before the journey;
4. a scripted step is still built: a missing required input, a failing provider and a failing constructor abort as without a script;
5. an unscripted step, and an attempt scripted to `run for real`, run the real step;
6. each scripted ending has the consequences of the outcome it stands for, and scripted contributions are committed only on success;
7. scripted emits and global metadata appear as the step's own events and entries;
8. a script that leaves out an attempt, an unscripted rescue step of a step that can fail on attempt 1, or, with `ensure all scripted`, an unscripted step, is reported before the journey, listing each;
9. each mistake in writing a script is reported before the journey;
10. a scripted run's record holds every attempt handed to its step's construction, with the inputs it was built with, and a rescue step's requested contributions;
11. a failing reporter recorder interrupts every call and keeps nothing, and setting `journey_id` interrupts that call and every later one.

## Alternatives considered

- **Keep test support in each language's tech spec.** Rejected: its rules are observable, and the same test would read differently, and could mean something different, in each language.
- **Leave the names entirely to each language.** Rejected: tests, the book and agents read best when one sentence works in every language.
- **Require the names.** Rejected: conformance cannot observe a name, so, as with the instance interface in proposal 0062, the specification suggests them and each language decides.
- **Make test support part of tier 1.** Rejected: a tier is the meaning of a workflow, and a language should be able to claim tier 1 before it offers test support.
- **A path listing every event.** Rejected: a new log line or hook would break every path test, while the path is meant to change only when the flow does.
- **A fake replacing every provider of a service, by key.** Rejected: a test could not give two steps' providers different fakes.
- **A fake given as one instance.** Rejected: it ignores the provider's scope, and a test could not make the provider fail.
- **A script that names its executor now.** Not now: only the local executor exists in tier 1 (see Open questions).
- **A fixed wording for mistakes in a test.** Rejected: what is named is what a test needs; the wording is each language's.
- **Repeating the cause on a failed journey's line.** Rejected: the `given up` line before it already carries the cause.

## Open questions

1. **Running a script on another executor.** A scripted run always uses the local executor of its mode. When the task and distributed executors are proposed, a later proposal decides whether a script can name the executor that plays it.
