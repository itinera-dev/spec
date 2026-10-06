# 0056: Everything in the data bag is a value

- Issue: [#56](https://github.com/itinera-dev/spec/issues/56)
- Tier: 1
- Amends: [0008](0008-steps.md), [0009](0009-running-a-workflow.md)

## Summary

Everything in the data bag is a **value**: data that can be serialized. That covers initial data, contributions from steps and hooks, and what input adapters supply. Functions, closures, function pointers, open connections, handles and other objects that are not data never enter the data bag. Tier 1 does not serialize anything; this rule says only what may be stored.

Amends proposal 0008 (item 19, "A contribution MAY be any value the language can hold") and proposal 0009 (item 2, "values MAY be any value the language can hold"), today in spec 4.6 point 1 and 5.1 point 2.

## Motivation

- **Executors that are not local.** A durable executor, or any executor that runs a journey outside one process, must store the data bag and restore it, which needs every value in it to be serializable. If tier 1 allows any value, workflows written for the local executor can hold values no other executor could ever run, and moving to such an executor would break them. The rule has to hold from the first tier, while there are no such workflows yet.
- **The separation of business rules and flow control.** A closure or a function in the data bag is behaviour passed around as data. A step could hand code to a later step, or to a hook, through the bag, which bypasses the declared flow that steps and policies are meant to keep visible.
- **The output.** The data bag is the journey's output (5.8). Callers, tooling and the conformance suite, whose values are JSON, all expect it to be data.

## Design

1. **A value** is data that can be serialized: null, booleans, numbers, strings, lists and objects (maps from string keys to values), and any type of the language that serializes to them, such as a record or a struct whose fields are values.
2. **Not values:** functions, closures, function pointers, and objects that stand for something outside the data, such as connections, handles, streams or locks.
3. **Everything in the data bag MUST be a value:** the instance's initial data, the contributions of steps and hooks, and what input adapters supply for a step's input.
4. **Nothing is serialized in tier 1.** Implementations need not serialize values. The rule says what may be stored, so that a later executor can serialize them. Engine events still carry keys only, never values (2.6).
5. **Implementations SHOULD make storing a non-value impossible to express**, for example through types. This joins the rules a language may make impossible to express (#54), under the tag `@non-value`.
6. **Where it can be expressed, it is detected when the value is handed over**, and the journey cannot go on:
   - initial data that is not a value: `run` refuses the instance before any journey, listing each key concerned;
   - an input adapter that supplies a non-value: the adapter has failed, so `input_adapter_failed` is emitted and the journey is aborted with `step could not be built`;
   - a step or hook contributing a non-value: contributing it is illegal, so the journey is aborted with the new abort reason `not a value`, naming the step, or the policy and hook, and the key. The contribute call ends the running step or hook with the executor's signal, as #55 defines for a reporter that fails.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0056-data-bag-values/`, with the amended scenarios of earlier proposals tagged `@proposal-0056`.

## Alternatives considered

- **Keeping "any value the language can hold".** Rejected: it lets local workflows depend on values no non-local executor can store, and behaviour can travel through the bag.
- **Restricting values only in the tier that brings durable executors.** Rejected: workflows written in earlier tiers would then break when moved to such an executor. The rule costs nothing to follow from the start.
- **Serializing every value in tier 1, to check it.** Rejected: it costs time on every contribution, and languages with types can check it before anything runs.

## Open questions

None.
