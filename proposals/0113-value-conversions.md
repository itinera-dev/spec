# 0113: Value conversions are calls into custom code

- Issue: [#113](https://github.com/itinera-dev/spec/issues/113)
- Tier: 1, as the capability `value-conversions`
- Amends: [0054](0054-how-custom-code-fails.md)

## Summary

Some languages turn an application's own objects into values, and values back into the type a request asks for, by calling code the application writes: a **value conversion**. Chapter 5, 5.10 lists every point where the executor calls custom code and says the list is exhaustive for tier 1, but it does not list these conversions. This proposal adds them to the list with their outcomes: a conversion that fails while a value is read means `wrong type`, and a conversion that fails while a value is captured means `not a value`. Either MAY carry the conversion's error, as a diagnostic that conformance does not check. Their cases form a capability, `value-conversions`, claimed by a language whose executor calls, while a journey runs, code the application supplies or customises to turn its objects into values, or values into its types, or to check them.

Amends proposal 0054, whose list of the points where custom code is called, kept in 5.10, did not name value conversions. In the chapters it changes 1.3, 1.8, 3.8, 4.2, 5.1, 5.10 and 5.11.

## Motivation

Implementers need to know what happens when application code fails at a point the specification does not name.

1. A value is data that can be serialized (chapter 1, 1.3). A language with a type system that knows this can store a value as it is and read it back by its exact type: it converts nothing, and no application code that can fail is called to store or read one.
2. Other languages cannot. In TypeScript, for example, an application's class becomes a value through a method it writes, and a request for that class reads the value back through a static method it writes; a request may also check a value with a guard function or a schema the application gives. A language on the JVM with a mapper such as Jackson, and Python with a library such as pydantic, are in the same position.
3. Each of these is custom code, called by the executor while a journey runs, and each can throw or report an error. Point 1 of 5.10 says all custom code fails the same way, "wherever this specification does not give that error another meaning", and point 3 says the list of calls is exhaustive. A conversion is not in the list, so the specification gave an implementer no outcome for it. Read literally, a conversion that throws while a step is built could be `step could not be built`, carrying the error, as a throwing constructor is, or `wrong type`, as a value that does not convert. Two implementations could choose differently, and a conformance case could not decide between them.

## Design

1. **Value conversions are custom code.** A **value conversion** is code the application supplies or customises, where its language needs it, which the implementation calls while a journey runs or when `run` is handed an instance's initial data or initial global metadata, that:
   1. turns one of the application's objects into a value when the object is captured: contributed, emitted as event data, set as global metadata, given in a reason's details, supplied by an input adapter, supplied by a service provider with an instance, or held in an instance's initial data or initial global metadata (point 5); or
   2. turns a value into the type asked for, or checks that a value is of that type, when a request, or an input adapter through its access to the data bag, reads it: a step's input, a hook's or rescue step's request, an input adapter's or a service provider's request for data from the workflow, or a read through an input adapter's access to the data bag (chapter 4, 4.2 point 8).
2. **A conversion that fails while a value is read means the value has the wrong type.** When a conversion throws, returns an error, or reports that the value does not match, the value MUST be treated as of the wrong type for that request, with exactly the consequences chapters 4, 6 and 7 give a value of the wrong type: the journey is aborted with `wrong type`, naming whom they name, or, for a read through an input adapter's access, the adapter is told the value is of another type and nothing is emitted.
3. **A conversion that fails while a value is captured means it is not a value.** When a conversion throws or returns an error, what was captured MUST be treated as not a value, with exactly the consequences chapters 2, 4 and 6 give something that is not a value at that point: `not a value`, or `step could not be built` for an input adapter's supply, naming whom they name.
4. **The error MAY travel as a diagnostic.** A conversion that reports that a value does not match has no error, and the abort carries none. When a conversion throws or returns an error, the abort MAY carry that error, in `journey_aborted` as its message and in the result, as 5.10 point 4 says of failing custom code. Whether it is carried MUST NOT change the abort reason or its details, and conformance does not check it. The outcome is about the data, as for every other `wrong type` and `not a value`; the error is there so that a fault in the application's own conversion code stays visible in the event stream.
5. **Initial data and initial global metadata.** Wherever a language converts them, a conversion that fails makes that key not a value: `run` MUST refuse the instance, listing the key, as chapter 3, 3.7 says, and the refusal MAY carry the conversion's error, as point 4 says. A conversion that fails is never a failure while the instance is created: other failures then, such as producing the journey ID, stay outside the executor (5.1 point 1). A conversion the application calls itself after the journey, such as reading the result's data bag, is outside the journey and this list.
6. **5.10 lists them.** The table in 5.10 point 2 lists value conversions, while a value is read and while one is captured, with the outcomes above, so the list stays exhaustive.
7. **Not excluded by 3.8.** A failing conversion happens at run time, so a language that makes non-values impossible to express (chapter 3, 3.8 point 1.4) MUST still follow these rules.
8. **The capability `value-conversions`.** The cases of this proposal need a language whose executor calls, while a journey runs, code the application supplies or customises to turn its objects into values, or values into its types, or to check them, so they form a capability (chapter 1, 1.8), `value-conversions`, claimed alongside the tier as chapter 5, 5.11 point 4 says.
   1. Such a language MUST claim it, and MUST pass the cases tagged `@capability-value-conversions`.
   2. Any other language, such as one that stores values as they are and reads them by their exact type, calling no conversion, does not claim it. It does not list this proposal in its manifest, and still claims tier 1, as a language that does not claim `testing` does with proposal 0108.

No abort reason, status or lifecycle is added. Every outcome above already exists; this proposal only says which one a failing conversion produces.

## Events

None added or changed. A conversion that fails while a value is read emits what a value of the wrong type emits, and one that fails while a value is captured emits what something that is not a value emits.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), with the cases of the proposal it amends, under `cases/tier-1/0054-how-custom-code-fails/` in the files `value-conversions-*.feature`, tagged `@capability-value-conversions` and `@proposal-0113`. They use converted types of the runner's own, one whose conversions throw or return an error and one whose conversion or check reports that a value does not match. The cases must check that:

1. a step's input whose conversion fails, and one whose check refuses the value, abort the journey with `wrong type`, naming the step and the key, and so does one an input adapter supplies whose conversion fails, after `input_adapter_supplied`, naming the adapter too; the abort caused by the check carries no error;
2. a hook's data from the workflow whose conversion fails, and a hook's optional data from the step whose check refuses the value, abort the journey with `wrong type`, naming the policy, the hook and the key, before the hook runs; the abort caused by the check carries no error;
3. a rescue step's data from the step it rescues whose check refuses the value aborts the journey with `wrong type`, naming the rescue step and the key, with no error;
4. an input adapter's request whose conversion fails, and a service provider's request whose check refuses the value, abort the journey with `wrong type`, naming the step, the adapter or the provider, and the key, before the adapter or the provider is called; the abort caused by the check carries no error;
5. a read through an input adapter's access to the data bag whose conversion fails is reported as another type: no event is emitted, nothing is aborted, and the input comes from the data bag;
6. a contribution, event data, global metadata, a reason's details and a service provider's data whose conversion fails each abort the journey with `not a value`, naming whom it names for something that is not a value;
7. an input adapter's supply whose conversion fails emits `input_adapter_failed` and aborts the journey with `step could not be built`;
8. initial data and initial global metadata whose conversion fails are refused by `run`, listing the key, with no event.

Whether an abort or a refusal caused by a conversion that throws or returns an error carries that error is not checked. No existing case changes.

## Alternatives considered

1. **Treat a failing conversion as any other failing custom code.** While a step is built, that is `step could not be built` with the error; in a hook, `hook failed`; in a running step, an abnormal termination. Rejected: the same bad value would have a different outcome depending on where it is read, and on whether the language converts at all. A language that reads by exact type reports it as `wrong type` everywhere.
2. **Always carry the error with `wrong type` and `not a value`.** Rejected: a check that refuses a value has no error to carry, and an error's message differs by language, so conformance could not check it. Point 4 lets an implementation carry it instead, unchecked.
3. **Leave it to each language.** Rejected: conformance could then not check a failing conversion, and 5.10 would no longer be exhaustive.
4. **Run the cases for every language, without a capability.** Rejected: a language that calls no conversion cannot make one fail, so it could neither pass the cases nor prove the rule impossible to express, since there is nothing to express.
5. **Never carry the error.** Rejected: a conversion that throws is often a fault in the application's own code, and an abort that dropped its error would hide that fault where Itinera promises to show it, leaving a bare `wrong type` to investigate.

## Open questions

None.
