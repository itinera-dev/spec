# 0024: Abnormal terminations are retried only when the step allows it

- Issue: [#24](https://github.com/itinera-dev/spec/issues/24)
- Tier: 1
- Amends: [0008: Steps](0008-steps.md)
- Amended by [#83](https://github.com/itinera-dev/spec/issues/83): what a failure, an abort and a refusal carry.

## Summary

An abnormal termination is retried only when the step's descriptor marks it retriable. Proposal 0008 treated it like a retriable failure by default.

## Motivation

An abnormal termination is an error the step did not anticipate. Its side effects may already have happened, for example a payment sent before the error, so retrying it by default is unsafe. A step author who knows that retrying an unexpected error is safe says so explicitly.

## Design

1. **`abnormal termination retriable`** is a setting of each step's descriptor, false unless the descriptor says so, with no workflow-wide default. It uses the same word as a failure's `retriable` flag.
2. **When it is false**, an abnormal termination MUST make the step `Failed` at once, and its failure hook MUST be called with the cause `abnormal termination`. By default, the journey fails.
3. **When it is true**, an abnormal termination MUST be retried like a retriable failure, within the same retry budget. When the budget is exhausted, the failure hook MUST be called with the cause `retries exhausted`.
4. **An abnormal termination never aborts the journey.** It always ends as a failure, at once or after its retries.
5. **This replaces** item 6 of proposal 0008, "By default it is treated like a retriable failure, since nobody knows whether trying again would help", and adds `abnormal termination` to the causes the failure hook may receive (item 10 of proposal 0008).

## Events

None beyond those of proposal 0008.

## Conformance

The conformance cases for this proposal must check that:

1. by default, an abnormal termination is not retried, even with a retry budget, and the failure hook receives the cause `abnormal termination`;
2. when the descriptor marks it retriable, an abnormal termination is retried within the budget;
3. a retriable abnormal termination that exhausts the budget reaches the failure hook with the cause `retries exhausted`.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0024-abnormal-termination-retriable/`, and the amended scenario of proposal 0008 under `cases/tier-1/0008-steps/`.

## Alternatives considered

- **Keep retrying abnormal terminations by default** (proposal 0008 as accepted). Rejected: an unanticipated error may have had side effects, so retrying must be explicit.
- **An "abort on abnormal termination" setting.** Rejected: an abnormal termination is a failure of the step, not an illegal state of the journey, so it never aborts.
- **Naming the setting "fail on abnormal termination"** (true by default) or "retry on abnormal termination". Rejected in favour of `abnormal termination retriable`, false by default, which reuses the word of a failure's `retriable` flag and keeps every switch off by default.

## Open questions

None.
