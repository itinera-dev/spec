# 0091: Input adapters may read the data bag

- Issue: [#91](https://github.com/itinera-dev/spec/issues/91)
- Tier: 1
- Amends: [0060](0060-input-adapters-attached-to-steps.md), [0010](0010-hooks-lifecycles-and-roles.md) and [0054](0054-how-custom-code-fails.md)

## Summary

An input adapter may read the data bag itself, read-only, in addition to the data from the workflow it declares. It declares this access like any other request. What it reads that way is its own business: a key it does not find, or a value of a type it did not expect, emits no event and aborts nothing, and the adapter decides what to return. Policies and their hooks are unchanged: they never receive the data bag.

Amends proposal 0060 (design points 0 and 3), proposal 0010 (what hooks may read, spec 6.6 point 5) and proposal 0054 (the rules a language may make impossible to express, spec 3.8). In the chapters it changes 3.4 point 3, 3.8, 4.2 point 3.1, 4.6 point 7 and 6.6 point 5.

## Motivation

An adapter is called once for each input of each step it is attached to, and it decides from the step's name and the key what to supply. Proposal 0060 lets it request data from the workflow, but only by keys and types fixed when it is declared. Some adapters need keys they can only compute when they are called:

1. a pricing adapter attached to several steps reads `price.<step name>`, or `<key>.override`, whichever exists;
2. an adapter that maps a step's inputs onto a nested naming scheme in the bag reads a key derived from the input's key;
3. an adapter that supplies a value only when some key is present, and returns nothing otherwise, without causing `optional_input_absent` for every input it does not serve.

None of these can be declared in advance, so today they cannot be written. An adapter is part of the workflow that declares it (spec 3.4 point 1), not code reaching it from outside like a policy, so letting it read its own journey's data does not weaken the isolation of steps or of policies.

## Design

1. **A new request: the data bag, read-only.** An input adapter MAY request read access to the data bag, as a declared request like the others of 3.4 point 3, resolved in the order the adapter declares its requests (4.2 point 3.1). An adapter that does not declare it does not receive it.
2. **What it shows.** For the call it is given to, it lets the adapter look up any key of the data bag and read its value as a type the adapter names. It shows the data bag the step is being built from: everything committed before this attempt. That includes the contributions of the hooks of earlier attempts, which are committed when each hook returns (6.7), but never the contributions of an earlier attempt that did not succeed, which were never committed (4.6).
3. **An adapter never changes the journey's data.** It only supplies values (3.4 point 3), so nothing it does through this access, or with what it reads, may change the data bag:
   - implementations SHOULD make writing through the access impossible to express, where their language allows it;
   - where the language allows a write to be expressed at all, it MUST NOT reach the data bag, nor change what any step, hook or adapter receives afterwards, for example because the access is a copy of the bag or a view that refuses writes;
   - every value the adapter reads is its own copy, under the rules for received data (6.6 point 6).
4. **Reads are the adapter's own.** A read through this access is not a request resolved by the executor:
   - a key that is not in the bag is reported to the adapter as absent; it emits no `optional_input_absent` and does not abort;
   - a value of another type than the one the adapter asked for is reported to the adapter as such; it does not abort with `wrong type`;
   - the adapter then decides what to do: return a value, return nothing, or fail. Failing is handled as today: `input_adapter_failed`, then `journey_aborted` with `step could not be built` (5.10).
5. **No new events.** Reads emit no events. What the adapter supplies is traced as today, by `input_adapter_supplied` naming the adapter, and a value it supplies is still checked against the type the step declared (`wrong type`, naming the adapter, 4.2 point 5).
6. **Valid only during the call.** The access is valid only while the adapter is being called for one input, like a step's handles (4.6 point 7). Used afterwards, it MUST NOT reveal anything of the data bag: every lookup reports the key absent. It does not fail, since a later use happens inside another call of the same journey, where a failure would abort it.
7. **Rules a language may make impossible to express (3.8).**
   - A new item: an input adapter writing through its access to the data bag, under the tag `@bag-write`.
   - Item 5 also covers an input adapter's access to the data bag used after its call, under the existing tag `@late-handle`.
8. **Unchanged.**
   - Declared data from the workflow keeps the rules of 6.6 point 3, with its events and aborts. An adapter may use both.
   - 6.6 point 5 becomes: "A hook of a policy MUST NOT be given the data bag itself." A policy reaches the workflow from outside, through roles; an adapter is part of the workflow.
   - Steps never receive the data bag.
   - The order of events for an input of an adapted step (4.2 and 2.8).

## Events

None new. `optional_input_absent`, `journey_aborted` with `required data missing` or `wrong type`, and their naming of the adapter, apply only to declared requests, never to reads through this access.

## Conformance

STEPS.md gains these sentences:

- `And the input adapter "<adapter>" requests the data bag`: the adapter declares read access to the data bag.
- `And the input adapter "<adapter>" returns "<key>" of type <type>, read from the data bag, for "<input>"`: when called for that input, the adapter reads that key through its access, and returns the value if it found one of that type, or nothing otherwise.
- `And the input adapter "<adapter>" writes "<key>" = <value> through its data bag access`: the adapter writes through its access, as far as the language allows, then goes on.
- `And the input adapter "<adapter>" keeps its data bag access when called for step "<step>", and returns "<key>" of type <type>, read through that kept access, for "<input>" when called for step "<step>"`.
- `Then step "<step>" was built for attempt <n> with input "<key>" = <value>`.

Cases, in `0060-input-adapters-attached-to-steps/`, tagged `@proposal-0091`, as FORMAT.md asks for an amendment's scenarios:

1. An adapter that reads a key computed for the step (`price.charge`) and returns it builds the step with that value.
2. An adapter that reads a key the bag does not hold, and returns nothing: the input is resolved from the bag, and the events are exactly those of a journey with no adapter event and no `optional_input_absent`.
3. An adapter that reads a key as another type than the bag holds is not aborted by the read; it returns nothing, and the input is resolved from the bag.
4. An adapter in a step's second attempt sees what the first attempt's retry hook contributed.
5. An adapter in a step's second attempt does not see what the failed first attempt contributed.
6. Tagged `@bag-write`: an adapter that writes through its access leaves the data bag unchanged, and the next step reads the original value.
7. Tagged `@late-handle`: an adapter that keeps its access from one call and reads through it in a later call, after an earlier step contributed the key, does not see that contribution.

## Alternatives considered

- **Declared requests only** (spec 3.4 point 3 today). Keeps every read in the event stream, but adapters that need keys computed at run time cannot be written.
- **Reads through the access follow the rules of declared requests**, emitting `optional_input_absent` and aborting on a missing required key or a wrong type. Rejected: the adapter cannot say whether a key it probes is required, and a probe that aborts the journey defeats the purpose.
- **Giving hooks of policies the same access.** Rejected: a policy is reusable across workflows and reaches a workflow only through roles; reading the bag would tie it to one workflow's keys.
- **A separate tag for an access used after its call.** Rejected: it is the same rule as a late contributor or step reporter, so it shares `@late-handle`.

## Open questions

None.
