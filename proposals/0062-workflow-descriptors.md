# 0062: Workflow descriptors, and executors independent of how workflows are written

- Issue: [#62](https://github.com/itinera-dev/spec/issues/62)
- Tier: 1
- Amends: [0009](0009-running-a-workflow.md)

## Summary

Separate what a workflow **declares** from what one **journey** holds, and keep executors independent of how workflows are written.

- **The workflow descriptor** is the workflow's declaration, and a **policy descriptor** describes one policy. Both are fixed and shared by every instance.
- **Executors MUST NOT couple to how a workflow or its instance was produced.** Any executor of a language can run any instance of that language. How this is abstracted is each language's choice.
- **The executor builds steps and policies; an instance never does.**
- A **suggested**, non-normative list of what an instance may provide to an executor.

Amends proposal 0009 (items 1 to 4 and 7, the workflow instance and step descriptors), and adds terms to chapter 1.

## Motivation

The specification mixes two lifetimes. Step descriptors, policies and adapters are declared once for the workflow, while the journey ID, the reporters and the data belong to one journey. It also says the executor asks the instance for "everything else it needs" without saying what that is, so nothing stops an executor from depending on how a particular instance was produced, for example by one language's macros, and refusing instances written by hand or by another tool.

The exact interface between an instance and an executor cannot be observed by conformance, so it belongs to each language. What the specification can say is that executors never depend on how a workflow was produced, and it can suggest what an instance usually provides.

## Design

1. **The workflow descriptor** is the declaration of a workflow (3.1): its name, its step descriptors in order (3.2), its workflow policy descriptors in declaration order, and its input adapters (#60). It holds descriptions, and a way to build each step and policy, never instances.
2. **A policy descriptor** describes one policy: its name, the hooks it defines, with what each requests and its execution mode, and a way to build an instance of the policy.
3. **Descriptors are fixed and shared.** A workflow descriptor MUST NOT change once built, is the same for every instance of the workflow, and is what the listing reads (3.1 point 2).
4. **Executors are not coupled to workflows.** An executor MUST NOT depend on how a workflow, its descriptor or its instance was produced: by a builder, by macros or annotations, or by hand. Any executor of a language MUST be able to run any instance of that language. How this abstraction is expressed, as an interface, a trait or a protocol, is each language's choice, recorded in its tech spec.
5. **The executor builds; the instance does not.** Steps, step policies and workflow policies are built by the executor from their descriptors, at the moments the specification sets (4.2 for steps; #58 for policies). An instance never builds, holds or orders steps or policies.
6. **Admission applies to every descriptor.** A workflow descriptor MUST have been checked as 3.7 requires before a journey starts, either when it is built or when the instance is handed to `run`.
7. **A suggested interface (not normative).** An instance usually provides to an executor: its workflow descriptor; the roles the workflow provides; its journey ID and its reporters (#61); its data bag; and three operations on its data: data for a step (through the step's input adapter, then the data bag), data for the workflow (from the data bag only), and committing a contribution, saying whether it replaced an earlier value. Each operation reports what happened, so the executor can emit the events and aborts the specification requires.
8. **Vocabulary.** Chapter 1 gains **workflow descriptor** and **policy descriptor**. **Workflow** keeps its meaning, the plan, and its declaration is its workflow descriptor.

## Events

See the design.

## Conformance

Not observable on its own. Conformance keeps checking the behaviour that follows from it.

## Alternatives considered

- **A normative instance interface, with its operations listed as requirements.** Rejected: conformance cannot observe it, so it belongs to each language's tech spec; the specification only suggests it.
- **Leaving executors free to depend on how instances are produced** (the current specification). Rejected: executors within one language could refuse each other's instances.

## Open questions

None.
