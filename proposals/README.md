# Proposals

Accepted proposals, the PRDs of Itinera's features: why each feature exists and what was decided. They are never specific to a language, and once accepted they are history; the current definition of every feature is the behaviour specification under [../spec/](../spec/). One file per proposal, named `NNNN-short-name.md` after the issue the proposal came from. A file here is accepted: it was merged after the evaluation described in [PROCESS.md](../PROCESS.md). New proposal files follow [TEMPLATE.md](TEMPLATE.md).

## Accepted

- [0002: Foundations](0002-foundations.md) (tier 1): step isolation, declarative workflows, policies and data flow. Accepted in [#18](https://github.com/itinera-dev/spec/pull/18).
- [0008: Steps](0008-steps.md) (tier 1): building, outcomes (success, failure retriable or not, skipped), contributions and identity. Accepted in [#23](https://github.com/itinera-dev/spec/pull/23).
- [0009: Running a workflow](0009-running-a-workflow.md) (tier 1): the workflow instance, journey IDs, step descriptors, the scan, retries and the result. Accepted in [#26](https://github.com/itinera-dev/spec/pull/26).
- [0010: Hooks, lifecycles and workflow roles](0010-hooks-lifecycles-and-roles.md) (tier 1): the order of hooks, the lifecycles they return, what they read and write, and workflow roles. Accepted in [#31](https://github.com/itinera-dev/spec/pull/31).
- [0011: Events](0011-events.md) (tier 1): reporters, the executor's dispatcher, the event stream, fixed fields, the engine catalogue and its order. Accepted in [#36](https://github.com/itinera-dev/spec/pull/36).
- [0012: The local executor](0012-local-executor.md) (tier 1): running a journey, statelessness between journeys, and the tier 1 capabilities. Accepted in [#37](https://github.com/itinera-dev/spec/pull/37).
- [0024: Abnormal terminations are retried only when the step allows it](0024-abnormal-termination-retriable.md) (tier 1, amends 0008). Accepted in [#25](https://github.com/itinera-dev/spec/pull/25).
- [0027: Data a step or hook receives is read-only](0027-received-data-read-only.md) (tier 1, amends 0002). Accepted in [#30](https://github.com/itinera-dev/spec/pull/30).
- [0032: Configuration errors found while running](0032-configuration-errors.md) (tier 1, amends 0009). Accepted in [#35](https://github.com/itinera-dev/spec/pull/35).
- [0040: Events record facts and decisions](0040-events-facts-and-decisions.md) (tier 1, amends 0011). Accepted in [#43](https://github.com/itinera-dev/spec/pull/43).
- [0041: Hooks read data from the workflow without input adapters](0041-hook-data-from-the-data-bag.md) (tier 1, amends 0010). Accepted in [#44](https://github.com/itinera-dev/spec/pull/44).
- [0042: Configuration errors are caught before a journey starts](0042-configuration-errors-before-the-journey.md) (tier 1, amends 0032 and 0012). Accepted in [#45](https://github.com/itinera-dev/spec/pull/45).
- [0049: Custom code that throws](0049-custom-code-that-throws.md) (tier 1, amends 0009 and 0011). Accepted in [#52](https://github.com/itinera-dev/spec/pull/52).
- [0054: How custom code fails](0054-how-custom-code-fails.md) (tier 1, amends 0049). Accepted in [#68](https://github.com/itinera-dev/spec/pull/68).
- [0055: A reporter that fails while a step or hook runs](0055-reporter-fails-while-custom-code-runs.md) (tier 1, amends 0011). Accepted in [#69](https://github.com/itinera-dev/spec/pull/69).

## Under evaluation

- [#3](https://github.com/itinera-dev/spec/issues/3): rescue steps (tier 4).
