# Vision

## The problem

Software mixes two kinds of decision. Business rules decide what the business does: whether a payment is accepted, whether an order can ship. Flow control decides what happens around that work: whether to retry a failed piece, whether to try a different strategy, whether to take a completely different path.

A flow decision can depend on the business, yet it is rarely a decision that belongs inside a single business unit. Decoupling, ports and adapters, and dependency injection all try to separate orchestration from business units. They mostly separate a unit from its dependencies, not from its fate. A service still catches a timeout and retries, falls back to another provider, or calls the next service itself, and each of those is a flow decision buried in business code.

Distributed computing makes this worse. Services are free to call whatever they want, flow decisions are spread across them, and there is no single place where a flow can be seen or controlled.

## What Itinera does

Itinera makes the separation structural instead of a matter of discipline.

- **Steps know nothing.** A step is a business unit. It asks for data without knowing where the data comes from, contributes data without knowing where it goes, and reports an outcome. It cannot see the workflow running it, because there is nothing for it to call.
- **The flow is visible in one place.** A workflow is an ordered list of steps plus the policies attached to it. Reading the definition tells you what will run.
- **Policies decide, the executor carries out.** After each outcome the executor calls a hook, a policy that returns what should happen next, or nothing to accept the default. Hooks cannot reorder steps; moving to another path means moving to another workflow, explicitly.
- **Everything is observable.** Once decisions are taken out of business code, a flow can only be understood from what the engine did. Every executor reports a standard stream of events.
- **One meaning, many languages.** This specification defines what a workflow means. Implementations in Rust, then TypeScript and the JVM, follow it, and a shared conformance suite checks that they do.

## Principles

1. The specification is the source of truth. Implementations follow it; they never quietly diverge.
2. A step never depends on the workflow, the data store, the executor or anything else outside itself.
3. Order is strict and visible. The list of steps you read is the order that runs.
4. Executors differ in capability, never in meaning.
5. Anything a step did not anticipate is abnormal, and abnormal is rare by design.
6. Documentation is readable with a screen reader: headings, lists and prose, never ASCII-art diagrams.

## Not goals

- Itinera is not a general job queue or message bus.
- Itinera is not a visual process language such as BPMN, although its workflows can be drawn from their definitions.
- Itinera does not offer branching inside a workflow. Branching is moving between workflows.
