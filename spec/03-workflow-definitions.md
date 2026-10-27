# 3. Workflow definitions and configuration errors

Specification 0.1. Written from proposals [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md), [0009](../proposals/0009-running-a-workflow.md), [0010](../proposals/0010-hooks-lifecycles-and-roles.md), [0012](../proposals/0012-local-executor.md) and [0032](../proposals/0032-configuration-errors.md), as amended by [0042](../proposals/0042-configuration-errors-before-the-journey.md), [0054](../proposals/0054-how-custom-code-fails.md), [0056](../proposals/0056-data-bag-values.md), [0058](../proposals/0058-building-policies.md), [0060](../proposals/0060-input-adapters-attached-to-steps.md), [0061](../proposals/0061-instance-carries-journey-id-and-reporters.md), [0062](../proposals/0062-workflow-descriptors.md), [0091](../proposals/0091-adapters-read-the-data-bag.md), [0095](../proposals/0095-services-provided-to-steps.md), [0098](../proposals/0098-rescue-step-adapters-and-providers.md), [0102](../proposals/0102-global-metadata.md), [0104](../proposals/0104-service-implementation-on-events.md), [0106](../proposals/0106-retry-budget-alone.md), [0113](../proposals/0113-value-conversions.md), [0116](../proposals/0116-duplicate-adapter-names.md) and [0121](../proposals/0121-read-only-copies.md), and from proposal [0003](../proposals/0003-rescue-steps.md).

How a workflow is declared, and what happens when the declaration is wrong. How each language expresses a declaration (a builder, annotations, attributes, macros) is decided in its tech spec; this chapter defines only what a declaration contains and how it is checked.

## 3.1 A workflow is declarative

1. A workflow's steps, their order, and the policies, input adapters and service providers attached to them MUST be fixed before any journey starts, whatever the language's syntax.
2. They MUST be listable without running anything: for each step, its name, its position, the names of its policies in declaration order, the name of its input adapter, if any, and the services it requests, by key in the order it declares them, each with the name of the provider that serves it (3.9). A step that has a rescue step (chapter 7) also shows, in its entry, a **rescue section**: the rescue step's name, the name of its input adapter, if any, and the services it requests, by key in the order it declares them, each with the name of the provider that serves it. The rescue section has no position, since the rescue step runs only when its step fails, and no policies, since a rescue step has none. A step without a rescue step has no rescue section.
3. A workflow declares:
   - its **name**;
   - its **step descriptors**, in order (3.2);
   - the **workflow policies** attached to it, by their policy descriptors (3.3);
   - its **input adapters**, and the steps each is attached to (3.4);
   - the **workflow roles** it provides (3.5);
   - its **service providers**, and the steps each is attached to (3.9).
4. **The workflow descriptor.** Together, these make up the workflow's declaration, its **workflow descriptor**. It holds descriptions, and a way to build each step and policy, never instances. It MUST NOT change once built, is the same for every instance of the workflow, and is what the listing reads.
5. **What belongs to each instance.** The journey ID, the reporters, the initial data and the initial global metadata belong to each workflow instance, not to the workflow descriptor (chapter 5, 5.1).

## 3.2 Step descriptors

1. A workflow provides one **step descriptor** per step, in the order the steps run. A step descriptor holds:
   - the step's **name**;
   - how to **build** the step, including the services it requests, by key and type (chapter 4);
   - its **retry budget**: the number of retries allowed after the first attempt, **0** unless the descriptor says otherwise;
   - **`retry on abnormal termination`**: whether an abnormal termination is retried while the retry budget lasts, **true** unless the descriptor says otherwise;
   - the **step policies** attached to the step, in declaration order;
   - its **input adapter**, if any (3.4);
   - its **rescue step**, if any: how to build it, and nothing else (chapter 7). A step with a rescue step has a retry budget of 0. An input adapter or service provider for the rescue step is attached to its name, `<step name>_rescue`, as to a step (3.4, 3.9).
2. There is no workflow-wide default for the retry budget or for `retry on abnormal termination`: each step states its own, so the declaration shows everything.
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
8. **A policy descriptor** describes one policy: its name, the hooks it defines, with what each requests and its execution mode, and a way to build an instance. Which hooks a policy defines is fixed by its descriptor, so `hook defined twice` is caught at admission, before any instance exists.
9. **A policy's definition is reusable; its instances are not.** The executor builds them, never shares them: workflow policies once per journey, step policies for every attempt (chapter 6, 6.1).

## 3.4 Input adapters

1. An **input adapter** is part of the workflow that declares it. It has a **name**, unique among the workflow's adapters, and is attached to one or more of the workflow's steps. Two adapters with the same name are the violation `duplicate adapter name`, whether or not they are attached to the same steps. An input adapter and a service provider MAY have the same name, since each name is unique only among its own kind (3.9 point 1).
2. **One adapter per step.** A step has at most one input adapter. Two adapters attached to the same step are the violation `step adapted twice`, and so is one adapter whose list of steps names the step more than once, since it is then attached to the step more than once. An adapter attached to a step the workflow does not have is the violation `input adapter for unknown step`. The name of a rescue step, `<step name>_rescue`, MAY be named as if it were a step (chapter 7, 7.1): a rescue step has at most one input adapter, two adapters attached to its name, or one adapter naming it more than once, are `step adapted twice`, and an adapter attached to `<step name>_rescue` when the step has no rescue step is `input adapter for unknown step`. One adapter MAY be attached both to a step and to its rescue step, as to any two steps. An adapter attached to a step serves that step only, never its rescue step.
3. **What it may request**, each as a declared parameter, and only what it declares: the name of the step being built, the key being resolved, data from the workflow, read from the data bag and never through an adapter, with the rules of chapter 6, 6.6 point 3, the journey ID, and read access to the data bag itself (chapter 4, 4.2 point 8). It does not request roles, receives no contributor and emits no events of its own: it only supplies values.
4. What an input adapter does when a step is built is defined in chapter 4, 4.2.
5. Adapters are declared on the workflow, never on the step: each workflow knows how to adapt its own data. There are no adapters for data from a step, and adapters never serve hooks.

## 3.5 Workflow roles

1. A workflow MAY provide any number of **roles**: interfaces of operations that the workflow's own code implements.
2. A policy, step or workflow, MAY request roles (chapter 6). A policy attached to a workflow that does not provide one of the roles it requests is the violation `role not provided`.
3. Steps MUST NOT request roles. Steps request services instead (3.9).

## 3.6 Configuration errors

1. A **configuration error** is anything wrong with how a workflow is put together or supplied, which its author is responsible for. They fall into two groups:
   - **errors in how the workflow is put together**, which need only its declaration and the executor: a hook defined twice for one step or one workflow, two steps with the same name, a policy requesting a role the workflow does not provide, two input adapters attached to one step, or one input adapter attached to it more than once, two input adapters with the same name, an input adapter attached to a step or rescue step that does not exist, a step with both a rescue step and a retry budget greater than 0, a step requesting a service no provider serves for it or one key twice, two providers of one key serving one step, or one provider attached to it more than once, two service providers with the same name, a service provider attached to a step or rescue step that does not exist, a provider of another type than a step requests, a provider of journey scope requesting the step's name or the attempt number, a step, hook, service provider or reporter in an execution mode the executor does not accept, initial data that is not a value, and initial global metadata that is not a value or that sets `journey_id`. They are caught before any journey (3.7);
   - **errors that can only show up during a journey**: a step or policy that cannot be built, required data missing, a value of the wrong type, something contributed or emitted that is not a value, a hook returning a lifecycle it may not return, a rescue step that skips, and global metadata set under the key `journey_id`. They abort the journey (chapters 4, 5, 6 and 7).
2. **Violations named in this specification:**

   | Violation | Meaning |
   |---|---|
   | `hook defined twice` | two policies attached to the same step, or to the same workflow, define the same hook |
   | `duplicate step name` | two steps in one workflow have the same name |
   | `role not provided` | a policy attached to the workflow requests a role the workflow does not provide |
   | `mode not accepted` | a step, hook, service provider or reporter runs in an execution mode the executor does not accept |
   | `step adapted twice` | two input adapters are attached to the same step, or one input adapter is attached to it more than once |
   | `duplicate adapter name` | two input adapters of one workflow have the same name |
   | `input adapter for unknown step` | an input adapter is attached to a step or rescue step the workflow does not have |
   | `rescued step retried` | a step has both a rescue step and a retry budget greater than 0 (chapter 7) |
   | `service not provided` | a step requests a service that no provider serves for it |
   | `service requested twice` | a step requests the same key twice among its services |
   | `service provided twice` | two providers of one key are both attached to the same step, or both attached to no step, or one provider is attached to the same step more than once |
   | `duplicate provider name` | two service providers of one workflow have the same name |
   | `service provider for unknown step` | a service provider is attached to a step or rescue step the workflow does not have |
   | `wrong service type` | the provider serving a step supplies another type than the step requests for that key |
   | `attempt request in journey scope` | a service provider of journey scope requests the name of the step being built or the attempt number |

## 3.7 Admission: refused before any journey

1. **When the workflow is built.** Errors in how a workflow is put together that need only its declaration MUST be reported when the workflow is built, by the compiler, the macros, the builder or the workflow's constructor. Implementations SHOULD report them at compile time where their language allows it.
2. **When the instance is handed to an executor.** `mode not accepted` depends on the executor, so it MUST be reported when the instance is handed to the executor's `run`, before the journey starts, if it was not already caught when the workflow was built. Initial data that is not a value is also refused then, listing each key concerned, and so is initial global metadata that is not a value or that sets `journey_id` (chapter 2, 2.11). An implementation that does not guarantee, when a descriptor is built, that it was checked MUST check it at this point.
3. **In both cases:**
   - no journey MUST start;
   - no event MUST be emitted;
   - the developer MUST receive every violation found, not only the first.

   `run` reports this as a refusal, not as a journey result (chapter 5). A workflow refused when it is built has no instance, so no journey ID exists; an instance refused by `run` already has its own.
4. **Errors found during a journey** abort it with their own abort reasons (chapter 5). Every event already emitted remains, and `journey_aborted` is the last event, naming the step concerned, if any, the abort reason and its details. The journey's result is aborted, with the same reason and details.

## 3.8 Rules a language may make impossible to express

1. Through its types or signatures, a language MAY make it impossible to write:
   1. a hook returning a lifecycle it may not return (chapter 6, 6.3);
   2. a policy attached to a workflow that does not provide a role it requests (3.5);
   3. a step, hook, service provider or reporter in an execution mode the executor does not accept (chapter 5, 5.11);
   4. something that is not a value in the data bag, in event data, in a reason's details, in global metadata or in the data a service provider supplies with an instance (chapter 4, 4.6 and 4.9; chapter 2, 2.9 and 2.11);
   5. a contributor or step reporter used after its attempt or hook has ended, an input adapter's access to the data bag or a service provider's means of emitting events used after its call, or a service kept by a step after its attempt (chapter 4, 4.6, 4.2 and 4.9);
   6. an input adapter writing through its access to the data bag (chapter 4, 4.2);
   7. a rescue step that skips (chapter 7, 7.4 point 4);
   8. a service provider of journey scope requesting the name of the step being built or the attempt number (3.9);
   9. a service provider returning nothing, neither an instance nor an error (chapter 4, 4.9).

   For such a language, the rules of this specification about these situations do not apply. A value conversion or a copy that fails (chapter 5, 5.10) happens at run time, so a language that makes something that is not a value impossible to write still follows the rules for it.
2. **Each exclusion is proven.** A language that makes a rule impossible MUST prove it in its own test suite: for every conformance scenario it excludes, a test shows that the forbidden code is rejected, for example by compiling it and expecting a compilation error. Its tech spec lists, for each excluded scenario, the test that replaces it.
3. **Conformance.** The scenarios that need these rules carry the tags `@invalid-lifecycle`, `@role-not-provided`, `@mode-not-accepted`, `@non-value`, `@late-handle`, `@bag-write`, `@rescue-skip`, `@journey-scope-request` and `@no-instance`. A language lists the tags it makes impossible in its `conformance.json`, and still claims the tier: the excluded rules cannot occur in it.

## 3.9 Service providers

1. A **service provider** is part of the workflow that declares it. It has a **name**, unique among the workflow's service providers, the **key** of the service it supplies, the **type** of what it supplies, a **scope**, `journey` or `attempt`, and MAY be attached to one or more of the workflow's steps.
2. **Which provider serves a step.** A provider serves a step only for a service the step requests. For each service a step requests, the provider that serves it is the provider of that key attached to that step, if there is one; otherwise, the provider of that key attached to no step, which serves every step that requests the key and has no provider of its own for it. A provider attached to a step that does not request its key is allowed, and is never called for that step.
3. **Violations.**
   1. A step requesting a key that no provider serves for it is the violation `service not provided`, and a step requesting one key twice among its services is `service requested twice`.
   2. Two providers of one key both attached to the same step, or both attached to no step, are the violation `service provided twice`, whether or not a step requests the key, and so is one provider whose list of steps names a step more than once, since it is then attached to that step more than once. Two providers with the same name are `duplicate provider name`; a provider MAY have the same name as an input adapter (3.4 point 1).
   3. A provider attached to a step the workflow does not have is the violation `service provider for unknown step`, and so is one attached to `<step name>_rescue` when the step has no rescue step (point 7).
   4. A provider supplying another type than a step it serves requests for that key is the violation `wrong service type`. A provider that serves no step, such as one attached to no step whose key every requesting step has its own provider for, is checked against no step.
   5. A provider of journey scope requesting the name of the step being built or the attempt number is the violation `attempt request in journey scope` (point 4).
4. **What it may request**, each as a declared parameter, and only what it declares: the journey ID, data from the workflow, read from the data bag and never through an adapter, with the rules of chapter 6, 6.6 point 3, and a means of emitting `journey_*` events (chapter 2, 2.9). A provider of `attempt` scope MAY also request the name of the step being built and the attempt number; one of `journey` scope MUST NOT, since it is called once, for whichever step needs it first. A provider does not request roles, receives no contributor and never receives the data bag itself: it only supplies services.
5. Providers are declared on the workflow, never on the step: the workflow decides which implementation each step uses and how long it lives, and a step never knows. What a provider does to make an instance is the workflow's business; when the executor calls it, and which instance it lends, are defined in chapter 4, 4.9.
6. Each service provider is synchronous or asynchronous.
7. **Rescue steps.** A rescue step (chapter 7) requests services as any step does. The name of a rescue step, `<step name>_rescue`, MAY be named as if it were a step when a provider is attached to steps, though the rescue step stays undeclared (chapter 7, 7.1).
   1. **Which provider serves it.** For each service a rescue step requests, the provider that serves it is the provider of that key attached to `<step name>_rescue`, if there is one; otherwise, the provider of that key attached to no step, as point 2 says for any step. A provider attached to the step it rescues serves that step only, never its rescue step.
   2. **Violations.** The violations of point 3 apply to a rescue step as to a step: two providers of one key attached to `<step name>_rescue`, or one provider naming it more than once, are `service provided twice`, a key that neither a provider attached to `<step name>_rescue` nor one attached to no step supplies is `service not provided`, and a provider of another type than the rescue step requests is `wrong service type`.
   3. **Scopes.** A provider of `attempt` scope serving a rescue step receives the rescue step's name and its attempt number, 1. A provider of `journey` scope attached to a step and to its rescue step makes one instance per journey, which they share; one attached to `<step name>_rescue` alone makes one for the rescue step; and for one attached to no step, the executor lends the rescue step the instance that provider made for the journey, or calls the provider for it if none was made yet (chapter 4, 4.9).

## 3.10 Checked by

In the conformance repository: [`0002-foundations/listing.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/listing.feature), [`0002-foundations/policies.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/policies.feature), [`0008-steps/identity.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0008-steps/identity.feature), [`0010-hooks-lifecycles-and-roles/roles.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0010-hooks-lifecycles-and-roles/roles.feature), [`0032-configuration-errors/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0032-configuration-errors), and the execution mode case of [`0012-local-executor/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0012-local-executor); the refusals are tagged `@proposal-0042`; [`0060-input-adapters-attached-to-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0060-input-adapters-attached-to-steps); the admission and listing cases of [`0095-services-provided-to-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0095-services-provided-to-steps); and the admission and listing scenarios tagged `@proposal-0098` in [`0003-rescue-steps/declaring.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0003-rescue-steps/declaring.feature), [`0003-rescue-steps/input-adapters.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0003-rescue-steps/input-adapters.feature) and [`0095-services-provided-to-steps/rescue-steps.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0095-services-provided-to-steps/rescue-steps.feature).
