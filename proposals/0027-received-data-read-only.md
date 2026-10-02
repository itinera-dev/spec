# 0027: Data a step or hook receives is read-only

- Issue: [#27](https://github.com/itinera-dev/spec/issues/27)
- Tier: 1
- Amends: [0002: Foundations](0002-foundations.md)

## Summary

Everything a step or a hook receives is read-only for it. Changing a received value never changes the data bag, any contribution, or what any other step or hook receives, and a contribution is captured when it is made. The only way to change the data bag remains a contributor.

## Motivation

Steps and hooks never see the data bag: they ask for data and the workflow resolves it. That separation is only real if what they receive cannot be used to change the bag behind the workflow's back. In a language with shared mutable objects, a step that modifies a list it received could otherwise change what a later step sees, invisibly and without any event.

## Design

1. **Received data is read-only.** Every value a step or a hook receives (a step's inputs, data from the workflow, data from the step, a failure's reason or details, an error, the journey ID) is read-only for it.
2. **Isolation.** Changing a received value, where the language allows it at all, MUST NOT change the data bag, the contributions of any step or hook, or what any other step or hook receives.
3. **Contributions are captured when made.** A value passed to a contributor MUST be captured at that moment: changing the object afterwards MUST NOT change what was contributed.
4. **Writing goes only through a contributor**, as proposals 0002 and 0008 already say, so every change to the data bag is visible in the events.
5. **How it is guaranteed is each language's choice**, recorded in its tech spec: for example through ownership and borrowing, or by giving each receiver its own copy or an immutable view.

## Events

None.

## Conformance

The conformance cases for this proposal must check that:

1. a step changing its input does not change what a later step receives, nor the journey's output;
2. a hook changing data it received changes nothing elsewhere;
3. a contribution is captured when it is made.

In a language where received values cannot be changed at all, these cases pass trivially. The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0027-received-data-read-only/`.

## Alternatives considered

- **Leaving isolation implicit.** Rejected: in languages with shared mutable objects it does not hold unless stated and tested.
- **Forbidding changes only to the data bag, not to contributions.** Rejected: a step could contribute an object and keep changing it afterwards, changing the bag without any event.

## Open questions

None.
