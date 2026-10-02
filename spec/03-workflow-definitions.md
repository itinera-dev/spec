# 3. Workflow definitions and configuration errors

Specification 0.1. Written from proposals [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md), [0009](../proposals/0009-running-a-workflow.md), [0010](../proposals/0010-hooks-lifecycles-and-roles.md), [0012](../proposals/0012-local-executor.md) and [0032](../proposals/0032-configuration-errors.md).

How a workflow is declared, and what happens when the declaration is wrong. How each language expresses a declaration (a builder, annotations, attributes, macros) is decided in its tech spec; this chapter defines only what a declaration contains and how it is checked.

## 3.1 A workflow is declarative

1. A workflow's steps, their order, and the policies and input adapters attached to them MUST be fixed before any journey starts, whatever the language's syntax.
2. They MUST be listable without running anything: for each step, its name, its position, the names of its policies in declaration order, and the keys that have an input adapter.
3. A workflow declares:
   - its **name**;
   - its **step descriptors**, in order (3.2);
   - the **workflow policies** attached to it (3.3);
   - its **input adapters** and the pairs of step and key each serves (3.4);
   - the **workflow roles** it provides (3.5);
   - its **reporters** (chapter 2);
   - optionally, a custom **journey ID generator** (chapter 5).

## 3.2 Step descriptors

1. A workflow provides one **step descriptor** per step, in the order the steps run. A step descriptor holds:
   - the step's **name**;
   - how to **build** the step (chapter 4);
   - its **retry budget**: the number of retries allowed after the first attempt, **0** unless the descriptor says otherwise;
   - **`abnormal termination retriable`**: whether an abnormal termination may be retried, **false** unless the descriptor says otherwise;
   - the **step policies** attached to the step, in declaration order;
   - the **input adapters** associated with the step's keys.
2. There is no workflow-wide default for the retry budget or for `abnormal termination retriable`: each step states its own, so the declaration shows everything.
3. **Names.** A step's name is a non-empty, case-sensitive string. It MUST be unique within its workflow; two steps with the same name are the violation `duplicate step name`.
4. **Execution mode.** Each step is synchronous or asynchronous.

## 3.3 Policies

1. A **policy** defines one or more hooks, and each hook it defines MUST be identified as the specific hook it implements.
2. A **step policy** defines only step hooks and is attached to steps. A **workflow policy** defines only workflow hooks and is attached to workflows. A policy MUST NOT define both.
3. Policies are reusable: a step policy MAY be attached to steps in any workflow, and a workflow policy to any workflow. A step policy runs in the context of the step it is attached to, so it names no step; a hook of a step policy MAY request the name of the step it acts on.
4. Any number of step policies MAY be attached to a step and any number of workflow policies to a workflow. Taken together, the policies attached to one step MUST define each step hook at most once, and those attached to one workflow MUST define each workflow hook at most once. A hook defined more than once for the same step or workflow is the violation `hook defined twice`.
5. A hook that no attached policy defines takes its default behaviour (chapter 6).
6. Policies MUST be evaluated in the order they are declared. In tier 1 the order has no effect on behaviour, since each hook is defined at most once; it is kept because later tiers may add hooks that run cumulatively, in that order.
7. Each hook is synchronous or asynchronous.

## 3.4 Input adapters

1. An **input adapter** is declared on the workflow and associated with one or more pairs of step and key. It belongs to the workflow that declares it, and MAY serve several of its steps.
2. What an input adapter does when a step's input is resolved is defined in chapter 4.
3. Adapters are declared on the workflow, never on the step: each workflow knows how to adapt its own data. There are no adapters for data from a step.

## 3.5 Workflow roles

1. A workflow MAY provide any number of **roles**: interfaces of operations that the workflow's own code implements.
2. A policy, step or workflow, MAY request roles (chapter 6). A policy attached to a workflow that does not provide one of the roles it requests is the violation `role not provided`.
3. Steps MUST NOT request roles.

## 3.6 Configuration errors

1. A **configuration error** is anything wrong with how a workflow is put together or supplied, which its author is responsible for. For example:
   - a hook defined twice for one step or one workflow;
   - two steps with the same name;
   - a policy requesting a role the workflow does not provide;
   - an input adapter for a step or key that does not exist;
   - a step, hook or reporter in an execution mode the executor does not accept;
   - a workflow instance handed to an executor after it has already run, or while it is running;
   - a step that cannot be built, required data missing, a value of the wrong type, and a hook returning a lifecycle it may not return. These can only be found while the journey runs, and are defined in chapters 4 and 6.
2. **Violations named in this specification:**

   | Violation | Meaning |
   |---|---|
   | `hook defined twice` | two policies attached to the same step, or to the same workflow, define the same hook |
   | `duplicate step name` | two steps in one workflow have the same name |
   | `role not provided` | a policy attached to the workflow requests a role the workflow does not provide |
   | `mode not accepted` | a step, hook or reporter runs in an execution mode the executor does not accept |

## 3.7 Admission: when configuration errors are caught

1. **When the workflow is built, wherever possible.** Errors in how a workflow is put together SHOULD be caught by the compiler, the macros or the builder, before any journey exists. Implementations SHOULD report them at compile time where their language allows it. When one is caught then:
   - no journey MUST start;
   - the journey ID generator MUST NOT be called;
   - no event MUST be emitted;
   - the developer MUST receive every violation found, not only the first.
2. **When the journey runs, at the latest.** An error in how the workflow is put together that was not caught when it was built MUST be found when the journey runs, before its first step. The journey MUST then be aborted with the abort reason `invalid configuration`, listing every violation found: `journey_started` is emitted, no step runs, and `journey_aborted` is the last event.
3. **Errors found partway through a journey**, such as required data missing for a later step, abort the journey with the abort reasons defined in chapters 4 and 6. Every event already emitted remains, and `journey_aborted` is the last event, naming the step concerned, if any, the abort reason and its details. The journey's result is aborted, with the same step, reason and details.
4. **A language claims exactly one** of the capabilities `build-time-configuration-check` and `run-time-configuration-check`, saying which of points 1 and 2 applies to the violations in 3.6. Its conformance runner runs the cases tagged with the capability it claims.

## 3.8 Checked by

In the conformance repository: [`0002-foundations/listing.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/listing.feature), [`0002-foundations/policies.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/policies.feature), [`0008-steps/identity.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0008-steps/identity.feature), [`0010-hooks-lifecycles-and-roles/roles.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0010-hooks-lifecycles-and-roles/roles.feature), [`0032-configuration-errors/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0032-configuration-errors), and the execution mode cases of [`0012-local-executor/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0012-local-executor).
