# 0049: Custom code that throws

- Issue: [#49](https://github.com/itinera-dev/spec/issues/49)
- Tier: 1
- Amends: [0009: Running a workflow](0009-running-a-workflow.md) and [0011: Events](0011-events.md)
- Amended by [#54](https://github.com/itinera-dev/spec/issues/54): how custom code fails.
- Amended by [#58](https://github.com/itinera-dev/spec/issues/58): workflow policies are built per journey, step policies per attempt.
- Amended by [#61](https://github.com/itinera-dev/spec/issues/61): the workflow instance carries its journey ID and its reporters.

## Summary

The executor guards every call it makes into custom code: nothing that custom code throws ever escapes `run`. What follows depends on what was called, and this proposal lists every such call in tier 1. Two cases the specification did not address are defined: a journey ID generator that fails is a refusal, and a reporter that throws while `journey_aborted` is being delivered is ignored.

## Motivation

Itinera calls code it did not write at many points: constructors, input adapters, steps, hooks, role operations, reporters, dispatchers and ID generators. The rules for what happens when they fail were spread across proposals 0008, 0009, 0010 and 0011, and two were missing. A journey ID generator is developer code and can fail before any journey exists, and a reporter can throw on `journey_aborted`, where aborting again would loop. Implementers need one rule that says what "throws" means in every language and what each failure does.

## Design

### What "throws" means

1. **Custom code** is any code the executor calls that the workflow's developer supplies: everything listed in item 3.
2. **Throwing** means any failure that escapes custom code: an exception, a panic, or an error returned through the language's signature, such as an `Err` in Rust, wherever this specification does not give that error another meaning. Implementations MUST catch every one of them.

### Every call is guarded

3. **Nothing custom code throws MAY escape `run`.** In tier 1, the executor calls custom code at exactly these points, with these outcomes:

   | Custom code | When it throws |
   |---|---|
   | the journey ID generator, custom or default | a refusal: no journey starts, no event is emitted, the workflow's reporters receive nothing, and `run` reports the generator's error message to the developer, as for admission |
   | a dispatcher given to the executor, while adding the journey's reporters | a refusal, as for the ID generator |
   | a step's constructor, or an input adapter for one of its inputs | the journey MUST be aborted with `step could not be built` |
   | a running step | an abnormal termination of the attempt; the journey is not aborted |
   | a hook, including a role operation the hook calls | the journey MUST be aborted with `hook threw` |
   | a reporter, or a dispatcher given to the executor while dispatching, on any event except `journey_aborted` | the journey MUST be aborted with `reporter threw` |
   | a reporter, or a given dispatcher, while `journey_aborted` is being delivered | the failure MUST be caught and ignored: the first abort reason stands, no further event is emitted, and delivery continues to the remaining reporters |

4. **The list is exhaustive for tier 1.** A later proposal that adds a call into custom code MUST add it to this list, with its outcome.
5. **Reporters and dispatchers SHOULD be written so that they never throw**, since a throw aborts the journey.
6. **Delivery when a reporter throws** (spec defect #47): the event a reporter threw on MUST still be delivered to every reporter after it; then `journey_aborted`, with the reason `reporter threw`, is delivered to every reporter except the one that threw.

### What this replaces

7. Proposal 0009 did not say what happens when the ID generator fails; item 3 defines it. Item 7 of proposal 0011 is extended by items 3 and 6 for reporters and dispatchers. Proposals 0008 and 0010 are restated, not changed.

## Events

No new events. A refusal produces none.

## Conformance

The conformance cases for this proposal must check that:

1. an ID generator that fails is a refusal, with no event, and `run` reports its message;
2. a given dispatcher that throws while reporters are added is a refusal, with no event;
3. a step whose input adapter fails aborts the journey with `step could not be built`;
4. a role operation that throws inside a hook aborts the journey with `hook threw`;
5. a given dispatcher that throws while dispatching aborts the journey with `reporter threw`;
6. a reporter that throws on `journey_aborted` changes nothing: the abort reason stays the first one, `journey_aborted` is the last event, and the other reporters still receive it once.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0049-custom-code-that-throws/`. Item 6 is checked by the scenario tagged `@spec-defect-0047`, and the other outcomes by the cases of proposals 0008, 0009, 0010 and 0011.

## Alternatives considered

- **An aborted journey when the ID generator fails.** Rejected: every event must carry the journey ID, and there is none.
- **Letting `run` throw.** Rejected: `run` never throws for anything inside a journey, and reports refusals instead.
- **Aborting again when a reporter throws on `journey_aborted`.** Rejected: aborting delivers `journey_aborted`, so it would loop.
- **A setting to ignore reporters that throw.** Not in tier 1.

## Open questions

None.
