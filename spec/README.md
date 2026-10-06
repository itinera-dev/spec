# Behaviour specification

The normative behaviour specification of Itinera: exactly what every implementation must do, in terms the conformance suite can observe. Its chapters are written from accepted proposals in [../proposals/](../proposals/), with every accepted amendment applied, and use the key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY as described in RFC 2119. When a chapter and a proposal disagree, the chapter is right.

Nothing here is specific to one language. How each language implements these chapters is described in that language's tech specs, in its own repository. See "The three kinds of document" in [PROCESS.md](../PROCESS.md).

## Version

These chapters are **specification 0.1.0**: tier 1, a single workflow, written from proposals 0002, 0008, 0009, 0010, 0011, 0012, 0024, 0027, 0032, 0040, 0041, 0042, 0049, 0054, 0055, 0056, 0057, 0058, 0060, 0061, 0062, 0063, 0064 and 0065, with the spec defects corrected so far. Until a first implementation passes every tier 1 conformance case, the cases are released as candidates (`v0.1.0-rc.N`); then both repositories are tagged `v0.1.0`. See "The behaviour specification" in [PROCESS.md](../PROCESS.md).

## Chapters

Read chapter 1 first. The others are ordered so that each builds only on the chapters before it, which is also an order in which they can be implemented.

1. [Concepts](01-concepts.md): what each concept is and how they relate. The single source of the vocabulary.
2. [Events](02-events.md): reporters, the dispatcher, the event stream, fixed fields, the engine catalogue and the order of events.
3. [Workflow definitions and configuration errors](03-workflow-definitions.md): step descriptors, policies, input adapters, roles, violations, and when configuration errors are caught.
4. [Steps](04-steps.md): building a step and resolving its inputs, outcomes, abnormal termination, skipping, contributions, and read-only data.
5. [Running a journey](05-running-a-journey.md): the workflow instance, journey IDs, the data bag, step statuses, the scan, retries, how a journey ends, the result, the executor and execution modes.
6. [Hooks, lifecycles and workflow roles](06-hooks-lifecycles-and-roles.md): which hooks run and in what order, what they may return, what they read and write, and roles.

## Suggested implementation order

This section is a suggestion, not part of the specification. Each language decides its own order in its tech specs. The order below gets a conformance runner asserting on events as early as possible, since every case observes a journey through its event stream.

1. **The harness.** A conformance runner that loads the cases, implements the step catalogue in the conformance repository, records events with a reporter, and writes its report. No case passes yet, but every later stage is checked as soon as it exists.
2. **Events** (chapter 2). The event type with its fixed fields, the reporter, the default dispatcher, and dispatcher factories.
3. **A minimal executor** (parts of chapters 3 and 5). A workflow with one synchronous step that succeeds, a workflow instance, journey IDs, and `run` returning a result. The first cases can then pass: most of `0011-events/reporters-and-dispatchers.feature`, the fixed-fields scenario of `0011-events/stream-and-catalogue.feature`, the journey ID scenarios of `0009-running-a-workflow/data-and-ids.feature`, and the statelessness scenarios of `0012-local-executor/`.
4. **Workflow definitions and configuration errors** (chapter 3). Step descriptors, attaching policies and input adapters, listing, violations and admission: `0002-foundations/listing.feature`, the admission scenarios of `0002-foundations/policies.feature`, `0008-steps/identity.feature`, and the admission scenarios of `0032-configuration-errors/` and `0012-local-executor/`, tagged `@proposal-0042`.
5. **Steps** (chapter 4). Building, input resolution, outcomes, contributions and read-only data: `0002-foundations/data-from-the-workflow.feature` and the scenarios of `0008-steps/` and `0027-received-data-read-only/` that use no hook.
6. **The scan, retries and the result** (chapter 5): `0009-running-a-workflow/` and the rest of `0012-local-executor/` and `0032-configuration-errors/`.
7. **Hooks, lifecycles and roles** (chapter 6): `0010-hooks-lifecycles-and-roles/`, `0041-hook-data-from-the-data-bag/`, `0040-events-facts-and-decisions/`, `0002-foundations/data-from-the-step.feature`, and every remaining scenario, many of which use a hook to observe what a step did.

A language adds a proposal to its `conformance.json` only when every case of that proposal passes, so most proposals complete at stage 7; before then, the runner can be pointed at single files or scenarios locally.
