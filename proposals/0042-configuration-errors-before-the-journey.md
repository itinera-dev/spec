# 0042: Configuration errors are caught before a journey starts

- Issue: [#42](https://github.com/itinera-dev/spec/issues/42)
- Tier: 1
- Amends: [0032: Configuration errors found while running](0032-configuration-errors.md) and [0012: The local executor](0012-local-executor.md)
- Amended by [#60](https://github.com/itinera-dev/spec/issues/60): input adapters are attached to steps, and leave unknown inputs to the data bag.

## Summary

Errors in how a workflow is put together are always caught before a journey starts: when the workflow is built, or, for an execution mode the executor does not accept, when the instance is handed to the executor. The capabilities `build-time-configuration-check` and `run-time-configuration-check`, and the abort reason `invalid configuration`, are removed.

## Motivation

Every check on how a workflow is put together needs only its declaration, and every language can make it when the workflow is built: at compile time, in the builder, or as the first thing a workflow's constructor does. A language that could only find a duplicate step name once a journey runs is hard to imagine, and supporting one costs a capability, an abort reason and a second set of cases. The only check that depends on something other than the declaration is the execution mode, which depends on the executor; it can still be made before the journey starts.

## Design

1. **Declaration violations are caught when the workflow is built.** `hook defined twice`, `duplicate step name`, `role not provided`, and an input adapter for a step or key that does not exist MUST be reported when the workflow is built, by the compiler, the macros, the builder or the workflow's constructor. Implementations SHOULD report them at compile time where their language allows it.
2. **`mode not accepted` is caught when the instance is handed to an executor**, before the journey starts, if it was not already caught when the workflow was built.
3. **In both cases:** no journey MUST start, the journey ID generator MUST NOT be called, no event MUST be emitted, and the developer MUST receive every violation found. `run` reports this as a refusal, not as a journey result: proposal 0012 forbids throwing only for what happens inside a journey.
4. **Configuration errors found during a journey are unchanged:** `step could not be built`, `required data missing`, `wrong type` and `invalid lifecycle` abort the journey, the events already emitted remain, and `journey_aborted` is the last event.
5. **Removed:**
   - the capabilities `build-time-configuration-check` and `run-time-configuration-check` (proposals 0032 and 0012). The tier 1 capabilities are `sync` and `async`;
   - the abort reason `invalid configuration` (proposal 0032). The abort reasons become `step could not be built`, `required data missing`, `wrong type`, `invalid lifecycle`, `hook threw` and `reporter threw`.
6. **This replaces** items 2 to 4 of proposal 0032 where they allow errors in how a workflow is put together to be found once the journey runs, and items 9 and 10 of proposal 0012 where they refer to those capabilities.

## Events

No new events. A refusal before a journey produces none.

## Conformance

The conformance cases for this proposal must check that:

1. a declaration violation is refused before any journey, with no event and no journey ID, listing every violation;
2. a synchronous-only executor refuses an asynchronous step before the journey starts, with no event and no journey ID.

The scenarios tagged `@capability-run-time-configuration-check` are removed, and the tag `@capability-build-time-configuration-check` is dropped. The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), tagged `@proposal-0042` under `cases/tier-1/0032-configuration-errors/` and `cases/tier-1/0012-local-executor/`.

## Alternatives considered

- **Keeping a run-time configuration check as a capability** (proposals 0032 and 0012 as accepted). Rejected: no language needs it, and it costs a capability, an abort reason and duplicate cases.

## Open questions

None.
