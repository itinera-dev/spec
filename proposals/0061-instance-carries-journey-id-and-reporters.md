# 0061: The workflow instance carries its journey ID and its reporters

- Issue: [#61](https://github.com/itinera-dev/spec/issues/61)
- Tier: 1
- Amends: [0009](0009-running-a-workflow.md), [0049](0049-custom-code-that-throws.md)

## Summary

The workflow instance provides its journey ID and its reporters, already made, when it is handed to the executor. Both are produced while the instance is created, by the developer's own code: first the ID, then the reporters, which may receive it. The executor no longer calls an ID generator. Anything that fails while the instance is created happens before the executor is involved, as the specification already says (5.1 point 1).

Amends proposal 0009 (items 5 and 6, journey IDs) and proposal 0049 (the ID generator row of its table), with the related text of 3.7, 5.2 and 5.9.

## Motivation

Today the executor calls the workflow's ID generator inside `run` (5.2 point 2), and a generator that throws is a refusal from `run` (5.2 point 4, 5.10). Reporters are built by the instance, which hands them to the executor (2.2 point 3, 5.1 point 3). So a reporter cannot know the journey ID when it is built: the ID does not exist yet. A reporter that wants it, for example to name a log file or a trace, has to wait for its first event.

Producing the ID when the instance is created puts everything a journey is made of in one place: its data, its ID and its reporters, ready before the executor is involved. It also gives the ID generator one simple rule. It is the developer's code that creates the instance, and the specification already places any failure there outside the journey.

## Design

1. **The instance provides its journey ID.** When the instance is handed to the executor, it already holds its journey ID, and the executor MUST use it. The executor never produces an ID and never calls a generator. The caller still does not pass an ID to `run` (5.2 point 3).
2. **How the ID is produced is the workflow's choice.** It is produced while the instance is created. Implementations MUST offer a default that produces a UUID v4, used unless the workflow provides its own way, which MAY derive the ID from the instance's data. An instance runs at most one journey (5.1 point 4), so each journey still has exactly one ID.
3. **The instance provides its reporters, already made.** They are made while the instance is created, after the journey ID, and MAY receive it. The executor asks the instance for them and adds them to its dispatcher (2.2 point 3), unchanged.
4. **Failures while the instance is created are outside the journey.** A failure in producing the ID or in making a reporter is a failure to create the instance (5.1 point 1). There is no journey, the executor is not involved, and the specification defines nothing more. The row "the journey ID generator, custom or default" is removed from the table of 5.10.
5. **Refusals from `run` and the journey ID.** A refusal from `run` still means no journey and no event. It no longer promises that no ID was produced, since the instance already has one. This applies to `mode not accepted`, a given dispatcher that throws while reporters are added, and any other refusal when the instance is handed over. 3.7 point 3 ("the journey ID generator MUST NOT be called") and 5.9 point 3 keep that promise only for refusals when the workflow is built, where no instance exists.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0061-instance-carries-journey-id-and-reporters/`, with the amended scenarios of earlier proposals tagged `@proposal-0061`.

## Alternatives considered

- **The executor calls the generator in `run`** (the current specification). Rejected: reporters cannot receive the ID when they are made, and a failing generator needs its own rule inside `run`.
- **Reporters made inside `run`, after the ID.** Rejected: a failure in making a reporter would then be another refusal inside `run`, and the instance would no longer hold everything its journey is made of.
- **Reporters taking the ID from `journey_started`.** Still possible, but it forces every reporter that needs the ID to wait for its first event.

## Open questions

None.
