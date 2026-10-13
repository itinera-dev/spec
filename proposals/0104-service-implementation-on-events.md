# 0104: The implementation a service provider made, and its data

- Issue: [#104](https://github.com/itinera-dev/spec/issues/104)
- Tier: 1
- Amends: [0011](0011-events.md) and [0095](0095-services-provided-to-steps.md)

## Summary

`service_provided` names the **implementation** the provider made: the concrete type of the instance, as the language names it at run time, and not the service type the step requested. A provider MAY also supply optional **data** with the instance, which the event carries for reporters. A reader of the event stream can then tell which implementation each step received, such as a sandbox bank or the real one, without the step knowing anything about it.

Amends proposal 0095, in what a provider returns and what `service_provided` carries. Amends proposal 0011, in the rule that engine events carry no data values, for the data a provider supplies. In the chapters it changes 2.6 point 4, 2.7, 3.8, 4.9 point 6 and the abort reasons of 5.7.

## Motivation

A workflow decides which implementation of a service each step uses, and a step never knows (4.9 point 2). Whoever reads a journey afterwards, in a log, a trace or an audit trail, often needs exactly that: did this payment go to the sandbox or to the real bank? Today `service_provided` names only the step, the attempt, the key and the provider. The provider's name hints at the implementation, but a provider may choose between implementations at run time, from configuration or from data in the data bag, and the stream never says which one it made.

The executor holds the instance when the provider returns it, so it can name the instance's type without asking the step or the application for anything. Some languages cannot always tell that type, so the rule says what to name in that case. A provider that wants to say more than a type name, such as which environment an instance points at, supplies data with it.

## Design

### The implementation

1. **Always carried.** `service_provided` MUST carry an **implementation**: a string naming the instance the provider returned.
2. **The concrete type.** The implementation SHOULD be the name of the instance's concrete type, as the language reports it at run time, and not the name of the service type the step requested. Two providers of one key that make instances of different types therefore produce different implementations.
3. **When a language cannot tell.** Where a language cannot tell an instance's concrete type, such as an object with no class of its own, the implementation is the name of the type the provider declares it supplies. The language's tech spec records when that happens.
4. **For reading, not matching.** The format of the name is the language's own, and MAY change between versions of the language or its toolchain. Reporters SHOULD NOT depend on it, and nothing in the specification compares it.
5. **Only the type.** A provider cannot replace the implementation with a name of its own. What it wants to add goes in its data.

### The provider's data

1. **Optional.** With an instance, a provider MAY supply **data**, for example `{"environment": "sandbox"}`. `service_provided` carries it, and carries no data when the provider supplies none.
2. **A value.** The data MUST be a value (4.6), as the data of a `journey_*` event (2.9 point 3). The event carries its own copy, captured when the provider returns. The executor MUST NOT serialize it: serializing belongs to reporters.
3. **For reporters only.** The data never enters the data bag or the result, and the step that receives the instance never sees it. No step, hook, input adapter or service provider can read it.
4. **Data that is not a value is illegal.** Where a language can express it, the journey is aborted with `not a value`, naming the step, the provider and the event kind `service_provided`. Neither `service_provided` nor `service_provider_failed` is emitted, the step is not given the instance, and its status becomes `Aborted` as for any step that cannot be built (4.2 point 7). An instance of journey scope made in that call is never lent.
5. **Engine events and data values.** Engine events still carry keys and never data values (2.6 point 4), except this data: like a reason or an error message, its author wrote it for the event.

### Unchanged

1. `service_provider_failed` carries neither an implementation nor data: no instance was made.
2. A step lent an instance of journey scope made for an earlier attempt causes no event, as before. The implementation and data appear once, on the `service_provided` of the call that made the instance.
3. The listing does not show implementations: a provider chooses its instance at run time.
4. How a language reads an instance's type, and how a provider supplies data, are each language's choice, recorded in its tech spec.

## Events

`service_provided` (2.7) carries, in addition to the step, the attempt, the key and the provider:

| Event | When | Also carries |
|---|---|---|
| `service_provided` | a service provider returned an instance for a step's service | step, attempt, key, provider, the implementation, and the data the provider supplied, if any |

`journey_aborted` for `not a value` already names the provider and the event kind when the value was the provider's; the event kind is now also `service_provided`.

## Conformance

Cases for this amendment live with the cases of proposal 0095, in `cases/tier-1/0095-services-provided-to-steps/`, tagged `@proposal-0104`. No existing case changes: event tables compare only the columns they list.

1. `service_provided` names the implementation the provider made: two providers of one key and one service type, making instances of two different implementations, give two different implementations, each the name the runner's language gives that implementation's type.
2. Data a provider supplies with its instance is carried by `service_provided`, as supplied.
3. A provider that supplies no data gives a `service_provided` with no data.
4. Tagged `@non-value`: data that is not a value aborts the journey with `not a value`, naming the step and the provider; neither `service_provided` nor `service_provider_failed` is emitted, the step is `Aborted` and never runs.

## Alternatives considered

- **A supertype every service type must extend, with an operation returning its own name.** Not chosen: the executor can read the name itself when the provider returns the instance.
- **An optional implementation, absent when the language cannot tell.** Rejected by the maintainer: reporters can always rely on the field, and naming the declared type is still useful.
- **Requiring every language to name the concrete type, by making the provider name it where the language cannot.** Rejected: the rule asks languages to try, and the tech spec records when they cannot.
- **Letting a provider name the implementation itself.** Rejected by the maintainer: the implementation stays one fact the provider cannot get wrong, and anything more is the provider's data.
- **A `journey_info` the provider emits before `service_provided`.** Possible today, but a reporter would have to connect two events; the data on the fact itself keeps one event per instance.

## Open questions

None.
