# 0122: Custom code that returns an unexpected result

- Issue: [#122](https://github.com/itinera-dev/spec/issues/122)
- Tier: 1
- Amends: [0054](0054-how-custom-code-fails.md), [0010](0010-hooks-lifecycles-and-roles.md) and [0009](0009-running-a-workflow.md)

## Summary

Chapter 5, 5.10 point 1 says custom code fails by throwing, or by returning an error through its signature. Many languages also let custom code return something its call does not allow: Java's `null`, JavaScript's `undefined` or `null`, or an object of the wrong kind. This proposal makes the rule strict: when the executor waits for a result and gets anything that is not one of the results the specification defines for that call, the journey is aborted with the new abort reason `unexpected result`, or, before a journey, `run` is refused. It also says which calls have a result at all: a call whose result is always nothing is void, and what it returns is never read.

Amends proposal 0054, whose list of the ways custom code fails, kept in 5.10, did not say what a return its call cannot return means, proposal 0010, whose workflow hooks become void, and proposal 0009, whose journey ID becomes a string that is not empty. In the chapters it changes 1.5, 2.7, 3.8, 4.2, 4.4, 4.9, 5.1, 5.2, 5.7, 5.10 and 6.3.

## Motivation

Executor authors need one answer for every call into custom code, and the specification gives it for throwing and for returned errors only. Today:

1. A step's run, or a rescue step's, that ends without reporting an outcome, for example by returning `null` in Java or `undefined` in JavaScript, is neither an outcome (chapter 4, 4.3 point 1) nor a failure (5.10 point 1). Nothing says what its attempt is.
2. A hook that returns something that is neither nothing nor a lifecycle, such as a string or a number, is not covered: chapter 6, 6.3 point 1 defines nothing and the lifecycles, and `invalid lifecycle` is about a lifecycle its kind may not return.
3. A dispatcher factory, or a policy's construction, that returns `null` instead of a dispatcher or a policy is not covered either.

Such a return is a programming mistake, not a business failure. Treating it as a failure of the call, such as an abnormal termination of a step, would make it retriable (proposal 0024), so the mistake would be retried and hidden instead of stopping the journey. The specification already aborts for mistakes of this kind, such as `wrong type` and `not a value`.

## Design

1. **Two shapes of call.** Every call into custom code has one of two shapes, and each language's tech spec MUST say, for every call, which shape it has and how the language expresses it:
   1. **Void:** a call whose result is always nothing. These are reporters, a dispatcher while it adds reporters or delivers an event, role operations, and, in tier 1, workflow hooks (`on workflow success` and `on workflow failure`). The executor MUST NOT read what a void call returns: whatever it returns, where the language lets it return anything, is ignored. A void call MUST NOT be given a result type that stands for nothing, such as an empty optional; it is void in the language's own terms.
   2. **With a result:** every other call: a step's run, a rescue step's, step hooks, the constructors of steps, rescue steps and policies, the dispatcher factory, input adapters, service providers and value conversions. Where the specification lets such a call return nothing, as for a step hook keeping the default (6.3 point 1), an input adapter that does not supply its input (4.2) and a service provider that returns nothing (4.9 point 6.2), the language MUST express nothing by one explicit form of its own, such as an enumeration's variant or an empty optional, which its tech spec names. A null-like value, such as Java's `null` or JavaScript's `null` or `undefined`, is never that form.
2. **An unexpected result.** A call with a result that returns something that is not one of the results the specification defines for it has returned an unexpected result. That includes a null-like value where nothing is allowed, a result of the right kind but malformed, such as a failure without a code or an outcome that is none of the three, and something that looks like a lifecycle but is not one of tier 1's. A null-like value and the value null are kept apart: where a language uses a null-like value as its form of the value null (1.3), as JavaScript does with `null`, a call whose result is a value, such as an input adapter, returns the value null by returning it, and that value is checked against the type asked for like any other, so for an input of type integer it is `wrong type`; anything that is neither the language's form of nothing nor a value, such as `undefined` in JavaScript, is an unexpected result. A call whose result is never a value, such as a step's run, a hook, a constructor, the dispatcher factory or a service provider, never returns the value null, so a null-like value from it is always an unexpected result. Where the specification already gives a malformed part a meaning, that meaning stays: a reason's details that are not a value are `not a value` (4.3 point 3), and a value conversion's results keep the meanings 5.10 gives them.
3. **Its consequence.** An unexpected result MUST abort the journey with the new abort reason `unexpected result`, whatever the consequence 5.10 gives the call when it fails, naming the code that returned it: the step, the rescue step, the policy and hook, the input adapter or the service provider, and the key where there is one. It is not an outcome and not an abnormal termination, so it is never retried. For a call made before the journey, such as the dispatcher factory or a workflow policy's construction, `run` is refused, naming the code. The abort or refusal SHOULD carry a message saying what was returned, which conformance does not check.
4. **What stays.** A throw and a returned error keep the meanings 5.10 gives them. `invalid lifecycle` stays for a tier 1 lifecycle that the hook's kind may not return. Since workflow hooks are void, a lifecycle returned from one is ignored: a later proposal that lets a workflow hook decide something changes its shape to one with a result, and every language then changes its workflow hooks formally.
5. **Impossible to express.** Chapter 3, 3.8 point 1 gains an item: a call with a result returning an unexpected result. A language whose types rule it out lists the tag in its `conformance.json` and proves it, as 3.8 point 2 asks.
6. **The journey ID is a non-empty string.** Chapter 5, 5.2 states that the journey ID MUST be a string that is not empty. Producing anything else, such as a null-like value, a number, a promise or future, any other object, or an empty string, MUST make creating the instance fail, as the language's own error: there is no journey and no event, as for any other failure while the instance is created (5.1 point 1). A language whose types rule it out lists it in 3.8 as impossible to express.
7. **Asynchronous calls.** An asynchronous call, a void one included, MUST be awaited: its result is what it resolves to, and a rejection is a throw, with the meaning 5.10 gives it. A void call's future is awaited for its completion and its rejection only: what it resolves to is never read, as for a synchronous void call. Where a language's types can show that a call resolves to something its call cannot return, implementations SHOULD make it impossible to express.

## Events

None added or changed. `journey_aborted` carries the new abort reason `unexpected result`, with the code that returned it in its details.

## Conformance

Scenarios tagged `@unexpected-result`, one for each call the cases can describe in neutral terms: a step and a rescue step that return no outcome, an asynchronous step that resolves to none, a step hook that returns something that is not a lifecycle or a null-like value, an input adapter that returns no result, a service provider that returns a null-like value, and a step and a policy built as a null-like value. Each checks `journey_aborted` with `unexpected result`, naming the code, and that nothing is retried; a workflow policy built as a null-like value checks that `run` is refused. Untagged scenarios check that a workflow hook's return, synchronous or asynchronous, is never read, and that an asynchronous workflow hook is awaited. A journey ID that is not a string is tagged `@non-string-journey-id`, and an empty one `@empty-journey-id`; both check that creating the instance fails with no event. A language whose types rule these out claims the tags as impossible.

## Alternatives considered

1. **Decide it in each language.** That leaves a behaviour the conformance suite could observe to each implementation, which the process forbids, and different languages would disagree.
2. **Treat every unexpected return as nothing.** That would make a step returning `null` neither fail nor succeed, and gives a hook's mistake the default's meaning silently.
3. **Give each call the consequence 5.10 gives its failure,** as first proposed. A step returning `null` would be an abnormal termination and be retried, hiding a programming mistake. Rejected for the strict rule.
4. **Let a hook that simply ends mean nothing,** as JavaScript returns `undefined`. Rejected: a step hook has a result, and must say keep the default explicitly; only void calls end without a result.

## Open questions

None.
