# 0111: Suggested application interface (not normative)

- Issue: [#111](https://github.com/itinera-dev/spec/issues/111)
- Tier: none, as a suggestion that adds nothing an implementation must do

## Summary

A non-normative "Suggested names" section, like chapter 5, 5.13 and chapter 8, 8.7, in each chapter whose concepts an application handles directly: what a hook decides, what an input adapter supplies, the kinds of events, attempts, requests, reasons and the result. Each section suggests the names and shapes an implementation offers to applications, so that Itinera reads the same in every language. Nothing in it changes behaviour, and no conformance case checks it.

## Motivation

The specification says what an implementation does, and leaves how an application writes it to each language's tech spec. That is right for what is specific to one language. But many choices are not specific to one language, and when each language makes them on its own, the same concept gets different shapes. A reader moving between languages, or an agent writing for several, then has to relearn each one.

The Rust implementation reworked its application interface for 0.1.0-beta.3, in [itinera-dev/itinera-rs#159](https://github.com/itinera-dev/itinera-rs/issues/159), to stop writing "nothing" where a decision has a name, and to stop leaving the meaning of a value to its position. This proposal suggests the language-neutral parts of that work, so that later languages start from them.

## Design

Every suggestion is a suggestion, not part of the specification, as chapter 5, 5.13 and chapter 8, 8.7 are. Names are given in words, and each language spells them in its own casing, such as `keep_default` in Rust and `keepDefault` in TypeScript and Kotlin. A language's tech spec says why when it keeps a different name or shape.

1. **What a hook decides** (chapter 6, 6.3). A hook returns a lifecycle, or nothing, and nothing means the default. It is suggested that:
   1. a hook returns a **decision**, never a bare absence, and the default is a decision of its own, **keep default**;
   2. there is one decision type for the hooks on step success, with `keep default`, `finish workflow` and `fail workflow`, and one for the hooks on step failure, on step retry and on step abnormal termination, with `keep default` and `fail workflow`, so that a lifecycle a hook may not return cannot be written where the language allows it (6.3 point 2);
   3. a decision can be asked what it does: whether it `keeps default` or `finishes workflow`, and, through `fails workflow`, the reason when it fails the workflow, so that a test of a hook reads the decision without matching on it.
2. **What an input adapter supplies** (chapter 4, 4.2 point 3). An input adapter returns a value or nothing, and nothing means the input is read from the data bag. It is suggested that the adapter returns **supplied**, which is either a **value** or **from data bag**, and that supplied has a typed reader: given a type, it gives the value when one was supplied and has that type, and nothing otherwise. A test of an adapter then compares the supplied value directly.
3. **Kinds of events** (chapter 2, 2.5, 2.7 and 2.9). It is suggested that an event's kind is a closed set, with one member for each kind of the engine catalogue and of the events from steps and hooks, which displays as the kind in snake_case, as 2.5 writes it, and can be read back from that text.
4. **Attempts** (chapter 5, 5.6 point 1). It is suggested that an attempt number is a type of its own, counted from 1, that says whether it `is first`, instead of a bare integer, wherever an application meets one: in hooks, providers, events and test support.
5. **Required and optional requests** (chapter 4, 4.2, and chapter 6, 6.6). It is suggested that the declaration of an input, or of any request, says whether it is required or optional, and that the type of the answer follows from it: a required request gives the value, and an optional one gives the value or its absence. An application then never handles an absence that cannot happen.
6. **Reasons** (chapter 6, 6.3 point 4). A reason is a code, an optional message and optional details. It is suggested that wherever an application gives a reason, in a step's failure or a hook's `fail workflow`, its code alone is accepted, so that the common case is written as the code.
7. **The result** (chapter 5, 5.8). The result's content is the specification's, and its shape the language's (5.8 point 3). It is suggested that the final status can be asked whether it `is succeeded`, `is failed` or `is aborted`, and gives its `failure`, with its `cause`, `reason` and `error`, or its `abort`, so that a caller reads what it needs without matching on the status.

What is specific to one language stays in that language's tech specs. For Rust, that is a result type alias for the language's error type, lifetimes on handles, the modules items live in, and handles recognised by their type in the macros.

The suggestions go into the chapters as sections of their own, each listing only its chapter's names: chapter 2, 2.13 (point 3), chapter 4, 4.11 (points 2 and 5), chapter 5, 5.14 (points 4 and 7) and chapter 6, 6.11 (points 1, 5 and 6).

## Events

None. Nothing in this proposal changes an event.

## Conformance

None. A suggestion is not observable, so the conformance suite does not check it, as it does not check 5.13 or 8.7.

## Alternatives considered

- **A single guide document outside the chapters.** It would separate each suggestion from the rule it serves, and fall out of date when a chapter changes.
- **Leaving it to each language's tech spec, pointing at Rust's.** Nothing would then tell a new language which choices are meant to be shared.

## Open questions

None.
