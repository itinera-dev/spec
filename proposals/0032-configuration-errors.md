# 0032: Configuration errors found while running

- Issue: [#32](https://github.com/itinera-dev/spec/issues/32)
- Tier: 1
- Amends: [0009: Running a workflow](0009-running-a-workflow.md)
- Amended by [#42](https://github.com/itinera-dev/spec/issues/42): configuration errors are caught before a journey starts.

## Summary

Defines configuration errors, when they are caught, and what the event stream and the result show when one is found while a journey runs. Adds the abort reason `invalid configuration`.

## Motivation

Many things can be wrong with how a workflow is put together or supplied, and the workflow author is responsible for all of them. Some can be caught before anything runs, others only while running. Developers need one rule for both, and operators need the event stream to show exactly what happened before the error and what the error was.

## Design

1. **A configuration error** is anything wrong with how a workflow is put together or supplied, which its author is responsible for. Examples: a hook defined twice for one step or workflow, two steps with the same name, a policy requesting a role the workflow does not provide, an input adapter for a step or key that does not exist, a step the executor cannot run, a step that cannot be built, required data missing, a value of the wrong type, a hook returning a lifecycle it may not return.
2. **Caught when the workflow is built, whenever possible**, by the compiler, the macros or the builder, before any journey exists. Then no journey MUST start, the ID generator MUST NOT be called, **no event MUST be emitted**, and the developer receives every violation.
3. **Always caught while the journey runs, at the latest.** A configuration error found while running MUST abort the journey:
   - every event already emitted remains: the stream is never rewritten;
   - **`journey_aborted` MUST be the last event**, stating the step concerned, if any, the abort reason and its details;
   - the journey's result is aborted, with the same step, reason and details.
4. **Abort reasons for configuration errors.** The abort reasons `step could not be built`, `required data missing`, `wrong type` and `invalid lifecycle` of proposal 0009 are configuration errors found while running. A new abort reason, **`invalid configuration`**, covers errors in how the workflow is put together, such as a hook defined twice, two steps with the same name or a missing role, when they are found only once the journey runs; it MUST carry every violation found.
5. **Not configuration errors:** a hook that throws (`hook threw`) and a reporter that throws (`reporter threw`) are faults while running; they also abort the journey, with the same rules for the event stream.
6. **The abort reasons of proposal 0009 become:** `step could not be built`, `required data missing`, `wrong type`, `invalid lifecycle`, `invalid configuration`, `hook threw`, `reporter threw`.

## Events

No new events. `journey_aborted` carries the step, the abort reason and its details.

## Conformance

The conformance cases for this proposal must check that:

1. a configuration error found partway through a journey keeps the events already emitted, and `journey_aborted` is the last event, naming the step and the reason;
2. a configuration error caught when the workflow is built produces no event and no journey ID, and the refusal lists every violation;
3. in a language that only finds errors in how a workflow is put together once a journey runs, the journey is aborted with `invalid configuration`, listing every violation.

Cases 2 and 3 are tagged with the capabilities `build-time-configuration-check` and `run-time-configuration-check`; a language claims exactly one. The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0032-configuration-errors/`.

## Alternatives considered

- **Emitting an event when a workflow is refused before any journey.** Rejected: there is no journey to report on; the refusal is returned to the developer.
- **Ending an aborted journey's stream with only `journey_aborted`.** Rejected: the stream records what happened and is never rewritten.

## Open questions

None.
