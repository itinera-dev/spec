# 0054: How custom code fails

- Issue: [#54](https://github.com/itinera-dev/spec/issues/54)
- Tier: 1
- Amends: [0049](0049-custom-code-that-throws.md)
- Amended by [#91](https://github.com/itinera-dev/spec/issues/91): input adapters may read the data bag.

## Summary

How custom code fails, in every language, and which rules a language may make impossible to express.

1. **All custom code fails the same way**: by throwing, or by returning an error through its signature. What differs is the consequence. A step can recover, so its failure is an abnormal termination; hooks, role operations, reporters and dispatchers cannot report an error and carry on, so their failure aborts the journey.
2. **Unrecoverable failures are outside the model.** A failure the language itself treats as unrecoverable, such as a panic in Rust, is not a throw: the journey stops as if the process had crashed.
3. **The abort reasons `hook threw` and `reporter threw` become `hook failed` and `reporter failed`.**
4. **A language MAY make some rules impossible to express**, through its types or signatures. The scenarios that need them are tagged, and a language excludes a tag only with a test that proves the rule cannot be expressed.

Amends proposal 0049, and through it the parts of 0008, 0010, 0011, 0012 and 0042 that these rules come from.

## Motivation

Proposal 0049 defines throwing as "an exception, a panic, or an error returned through the language's signature" and requires implementations to catch every one. That does not fit every language. In Rust a panic is a bug, not an error: the language did not design it to be caught like an exception, and catching it with `catch_unwind` to keep a journey going turns it into a try/catch the language deliberately lacks. A panic is like an unhandled syntax error in PHP or JavaScript: beyond what Itinera can handle. A host that wants to contain it, for example by running each journey in its own task, does so at its own boundary.

At the same time, every kind of custom code needs a way to say "something went badly wrong, and I could not do what I was designed to do". In a language with exceptions that is a throw; in a language without them, it is an error returned through the signature. For a hook or a reporter, which cannot report an error and recover, that failure aborts the journey.

Finally, the specification already prefers rules that cannot be broken: it asks implementations to make invalid lifecycles impossible to express (6.3), to detect missing roles at compile time (6.8), and to make running an instance twice impossible to write (5.9). But a language that does so cannot pass the conformance scenarios that check those rules at run time: they assert a result, such as an abort with `invalid lifecycle`, that can no longer happen.

## Design

1. **How custom code fails.** Throwing (5.10 point 1) means any **recoverable** failure that escapes custom code: an exception, or an error returned through the language's signature, wherever the specification does not give that error another meaning. Every kind of custom code fails this way. The consequence depends on what failed (5.10): an abnormal termination for a running step, which its descriptor may retry; `step could not be built` for a step's constructor or input adapter; an abort for a hook, a role operation, a reporter or a dispatcher; a refusal before the journey.
2. **Unrecoverable failures.** In a language that separates unrecoverable failures from recoverable ones, as each language defines them (for example a panic in Rust, or an `Error` such as `OutOfMemoryError` on the JVM), an unrecoverable failure is not a throw.
   - The journey stops as if the process had crashed: no further event is emitted, so the event stream may end without `journey_succeeded`, `journey_failed` or `journey_aborted`, and `run` produces no result.
   - Implementations MUST NOT catch it in order to continue the journey or turn it into an outcome. It propagates to whoever called `run`, as the language defines.
   - "A panic" is removed from 1.2 (abnormal termination), 4.4 point 1, 5.10 point 1 and 6.5 point 1, and replaced by this rule.
3. **Abort reasons.** `hook threw` becomes `hook failed`, and `reporter threw` becomes `reporter failed`, everywhere: they cover both a throw and a returned error.
4. **Rules a language MAY make impossible to express.** Through its types or signatures, a language MAY make it impossible to write:
   1. a hook returning a lifecycle it may not return (6.3);
   2. a policy attached to a workflow that does not provide a role it requests (6.8);
   3. a step, hook or reporter in an execution mode the executor does not accept (5.11);
   4. a value that is not a value, in the sense of #56, in the data bag, in event data or in a reason's details (#56, #64);
   5. a contributor or step reporter used after its attempt or hook has ended (#57).

   For such a language, the rules of the specification about these situations do not apply.
5. **Each exclusion is proven.** A language that makes a rule impossible MUST prove it in its own test suite: for every scenario it excludes, a test shows that the forbidden code is rejected, for example a test that compiles it and expects a compilation error. Its tech spec lists, for each excluded scenario, the test that replaces it.
6. **Conformance tags.** Every scenario that needs one of these rules carries a tag, one per rule in point 4: `@invalid-lifecycle`, `@role-not-provided`, `@mode-not-accepted`, `@non-value` and `@late-handle`.
7. **The manifest.** A language lists the tags it makes impossible in `conformance.json`, under `impossible`. The `run-conformance` action excludes them, as it excludes capabilities the language does not claim. The conformance report lists each excluded scenario as not applicable, with the test that proves it.
8. **Claims.** A language that excludes these tags still claims the tier: the excluded rules cannot occur in it.

## Events

See the design.

## Conformance

1. The abort reasons `hook threw` and `reporter threw` are renamed in every case and in STEPS.md.
2. The scenarios that need a rule of point 4 carry its tag, including "A lifecycle a hook may not return aborts the journey" (`@invalid-lifecycle`), "A policy needing a role the workflow does not provide is refused at admission" (`@role-not-provided`) and "A synchronous-only executor refuses an asynchronous step before the journey starts" (`@mode-not-accepted`).
3. FORMAT.md documents the tags and the manifest's `impossible` list.

Unrecoverable failures are not checked by conformance: by definition, nothing observable follows them.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0054-how-custom-code-fails/`, with the amended scenarios of earlier proposals tagged `@proposal-0054`.

## Alternatives considered

- **Catching panics in Rust with `catch_unwind`** (proposal 0049 as accepted). Rejected: a panic is a bug, not an error; catching it uses unwinding as a try/catch the language does not offer, and does nothing under `panic = "abort"`.
- **Hooks, role operations and reporters that cannot fail at all, in languages without exceptions.** Rejected: such code needs a way to say something went badly wrong; without one, its only options are to swallow the error or to panic.
- **Letting the scenarios pass trivially.** Not possible: they assert results that cannot happen.
- **Moving these scenarios to each language's own tests.** Rejected: languages that can express these situations need them checked by the shared suite.
- **A second, untyped API in each such language, only so the shared scenarios can run.** Rejected: it would make the invalid states representable again.
- **Excluding without proof.** Rejected: a language could exclude a rule by mistake.

## Open questions

None.
