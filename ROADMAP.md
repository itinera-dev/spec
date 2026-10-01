# Roadmap

What is planned, in order. Status of individual proposals lives in their issues; this page gives the overall direction.

## Tiers

Tiers are cumulative: each includes everything before it. See [PROCESS.md](PROCESS.md) for how tiers and capabilities are claimed.

1. **Tier 1: a single workflow.** Steps, the data bag, a local executor, retries, step hooks with their default behaviour, the workflow result, workflow run IDs and the event model. Proposal: [#2](https://github.com/itinera-dev/spec/issues/2).
2. **Tier 2: transfers and switches.** Moving a journey to another workflow, with or without its data and execution state, late joining, transfer limits and re-entrancy protection. Recorded in [#2](https://github.com/itinera-dev/spec/issues/2).
3. **Tier 3: sub workflows.** Running a workflow from within another. Recorded in [#2](https://github.com/itinera-dev/spec/issues/2); data flow and sub journeys still open.
4. **Tier 4: rescue steps.** A step that fixes the cause of a failure, after which the failed step runs again. Proposal: [#3](https://github.com/itinera-dev/spec/issues/3).
5. **Later tiers.** Durable runs (pending steps, checkpointing, resuming after a crash), services provided to steps, and others.

## Milestones

1. **Foundations.** Organisation, licences, process, organisation profile, the core model proposal.
2. **Specification 0.1.** Tier 1 written into the specification; the conformance case format and the first cases.
3. **itinera-rs, tier 1.** Core types, the step, workflow and policy macros, local executors, a conformance runner, and the first release on crates.io.
4. **Tiers 2 to 4**, specified and implemented in Rust.
5. **Durable runs**, with a store interface and a first database-backed store.
6. **Studio**, which draws workflows and the connections between them from their definitions.
7. **More languages:** TypeScript, then the JVM (Kotlin, with support for Java).
