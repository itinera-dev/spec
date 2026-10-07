# 0065: The journey result is a business outcome

- Issue: [#65](https://github.com/itinera-dev/spec/issues/65)
- Tier: 1
- Amends: [0009](0009-running-a-workflow.md)
- Amended by [#83](https://github.com/itinera-dev/spec/issues/83): what a failure, an abort and a refusal carry.
- Amended by [#85](https://github.com/itinera-dev/spec/issues/85): an aborted journey's result carries no data bag, and an abort is final.

## Summary

The journey result is a business outcome: the journey ID, the final status with what explains it, and the data bag. It no longer carries the internals of the journey: every step's status and attempt count, and which step failed or was being run when the journey was aborted. Those stay observable where they already are, in the event stream, which records every step status (2.6 point 3), and `journey_failed` and `journey_aborted`, which name the step.

Amends proposal 0009 (items 17 to 19, the journey result), spec 5.8.

## Motivation

The code that runs a journey is business code: it wants to know whether the journey succeeded, why not if it did not, and what data it produced. Steps are how the workflow is put together, which the specification otherwise keeps away from callers. A result that lists every step's status and attempt count:

- couples callers to the workflow's internal structure, so renaming, splitting or reordering steps breaks code that only wanted the outcome;
- duplicates what the event stream already records, completely and in order, for whoever needs to observe the journey: operators, tests, and the conformance suite.

## Design

1. **The result MUST carry:**
   1. the **journey ID**;
   2. the **final status**, with what explains it:
      - **succeeded**;
      - **failed**, with why: the reason of the failure that failed the journey (code, message, details), or the cause `retries exhausted` with the reason of the last failure, or the error message of an abnormal termination, or the reason a hook gave with `FailWorkflow`;
      - **aborted**, with the abort reason and its details;
   3. the **data bag** as it stood at the end: the initial data plus every committed contribution. It is the journey's output.
2. **Not in the result:** the status and attempt count of each step, and the name of the step that failed or during which the journey was aborted. They are observable in the event stream: `attempt_started`, the outcome facts and the decisions give every step's status and attempt count (2.6 point 3, 2.7), and `journey_failed` and `journey_aborted` name the step.
3. **Implementations MAY offer** a way to derive step statuses from a journey's event stream, for example in a test helper. It is not part of the result.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0065-business-result/`, with the amended scenarios of earlier proposals tagged `@proposal-0065`.

## Alternatives considered

- **Keeping step statuses in the result** (the current specification). Rejected: it exposes the workflow's structure to business callers and duplicates the event stream.
- **Keeping the failed step's name in the result.** Rejected for the same reason: a step's name is internal, and `journey_failed` already carries it for whoever observes the journey.

## Open questions

None.
