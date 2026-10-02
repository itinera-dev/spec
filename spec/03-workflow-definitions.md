# 3. Workflow definitions and configuration errors

Specification 0.1. Written from proposals [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md), [0009](../proposals/0009-running-a-workflow.md), [0010](../proposals/0010-hooks-lifecycles-and-roles.md), [0012](../proposals/0012-local-executor.md) and [0032](../proposals/0032-configuration-errors.md), as amended by [0042](../proposals/0042-configuration-errors-before-the-journey.md).

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

1. A **configuration error** is anything wrong with how a workflow is put together or supplied, which its author is responsible for. They fall into two groups:
   - **errors in how the workflow is put together**, which need only its declaration and the executor: a hook defined twice for one step or one workflow, two steps with the same name, a policy requesting a role the workflow does not provide, an input adapter for a step or key that does not exist, and a step, hook or reporter in an execution mode the executor does not accept. They are caught before any journey (3.7);
   - **errors that can only show up during a journey**: a step that cannot be built, required data missing, a value of the wrong type, and a hook returning a lifecycle it may not return. They abort the journey (chapters 4, 5 and 6).
2. **Violations named in this specification:**

   | Violation | Meaning |
   |---|---|
   | `hook defined twice` | two policies attached to the same step, or to the same workflow, define the same hook |
   | `duplicate step name` | two steps in one workflow have the same name |
   | `role not provided` | a policy attached to the workflow requests a role the workflow does not provide |
   | `mode not accepted` | a step, hook or reporter runs in an execution mode the executor does not accept |

## 3.7 Admission: refused before any journey

1. **When the workflow is built.** Errors in how a workflow is put together that need only its declaration MUST be reported when the workflow is built, by the compiler, the macros, the builder or the workflow's constructor. Implementations SHOULD report them at compile time where their language allows it.
2. **When the instance is handed to an executor.** `mode not accepted` depends on the executor, so it MUST be reported when the instance is handed to the executor's `run`, before the journey starts, if it was not already caught when the workflow was built.
3. **In both cases:**
   - no journey MUST start;
   - the journey ID generator MUST NOT be called;
   - no event MUST be emitted;
   - the developer MUST receive every violation found, not only the first.

   `run` reports this as a refusal, not as a journey result (chapter 5).
4. **Errors found during a journey** abort it with their own abort reasons (chapter 5). Every event already emitted remains, and `journey_aborted` is the last event, naming the step concerned, if any, the abort reason and its details. The journey's result is aborted, with the same step, reason and details.

## 3.8 Checked by

In the conformance repository: [`0002-foundations/listing.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/listing.feature), [`0002-foundations/policies.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/policies.feature), [`0008-steps/identity.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0008-steps/identity.feature), [`0010-hooks-lifecycles-and-roles/roles.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0010-hooks-lifecycles-and-roles/roles.feature), [`0032-configuration-errors/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0032-configuration-errors), and the execution mode case of [`0012-local-executor/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0012-local-executor); the refusals are tagged `@proposal-0042`.
