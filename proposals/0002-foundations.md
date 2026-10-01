# 0002: Foundations

- Issue: [#2](https://github.com/itinera-dev/spec/issues/2)
- Tier: 1
- Accepted: YYYY-MM-DD

## Summary

The premises every Itinera implementation is built on, in every language: what a step can access, how a workflow declares its steps, how policies are attached and composed, and how data flows between the workflow, its steps and its hooks.

## Motivation

Separating business rules from flow control fails when the separation depends on discipline. A step author who can reach the workflow will, sooner or later, make a flow decision inside business code. A workflow assembled at run time cannot be read, checked or drawn before it runs. A decision written into one step cannot be reused by another.

These premises make the separation structural:

- **step authors** write business units that have nothing to call: data arrives through declared inputs, and leaves through a contributor;
- **workflow authors** declare the whole flow in one place, attach reusable policies where decisions are needed, and decide where each step's data comes from;
- **tooling and executors** can read a workflow without running it, and reject a broken one before anything starts.

## Design

### 1. Steps have no access to the workflow

A step MUST be able to access only what it declares it needs:

- data from the workflow, its inputs;
- a contributor, for the data it contributes back;
- a reporter, for its own events.

A step MUST NOT have access to the workflow, the data bag, the executor, other steps, or anything else that runs it.

### 2. Workflows are declarative

A workflow's steps, their order, and the policies and adapters attached to them MUST be fixed before the run starts, and MUST be listable without running anything, whatever the language's syntax.

### 3. Policies

A policy is an object that defines one or more hooks. Each hook it defines MUST be identified as the specific hook it implements. There are two types of policy:

- **step policies** define only step hooks, and are attached to steps;
- **workflow policies** define only workflow hooks, and are attached to workflows.

A policy MUST NOT define both step hooks and workflow hooks.

Policies are reusable. A step policy MAY be attached to steps in any workflow, and a workflow policy to any workflow. A step policy runs in the context of the step it is attached to, so it names no step; it MAY request the name of the step it is acting on.

### 4. Each hook is defined at most once per target

Any number of step policies MAY be attached to a step. Taken together, they MUST define each step hook at most once. The same holds for the workflow policies attached to a workflow, for workflow hooks.

A hook that no attached policy defines takes its default behaviour.

A hook defined more than once for the same target is a configuration error. It MUST be reported at admission, before anything runs, together with every other violation. Implementations SHOULD report it at compile time where their language allows it.

### 5. Declaration order

Policies MUST be evaluated in the order they are declared. In tier 1 the order has no effect on behaviour, since each hook is defined at most once. It is preserved because later tiers may add cumulative hooks, which will run in that order.

### 6. Data from the workflow, through input adapters

A step requests each input from the workflow by a key, local to the step, and a type.

1. An **input adapter** is declared on the workflow and associated with one or more pairs of step and key. When the step is built, the adapter receives the step's name and the key, MAY obtain the data from anywhere, and MUST return the value in the requested type.
2. Input adapters belong to the workflow that declares them, and MAY serve several of its steps.
3. When no input adapter is declared for a step and key, the value MUST be read from the workflow's data bag under that key.
4. If an adapter fails, or the value is missing or of the wrong type, the journey MUST be aborted. A step that cannot be built makes the run unsafe.

### 7. Data from the step, without adapters

Only step hooks receive data from a step.

1. A step hook requests data from the step it acts on by a key and a type, and MUST receive exactly the value that step contributed under that key.
2. There is no adapter in this direction, so the data is guaranteed to come from the step. Any transformation happens inside the hook.
3. If the step did not contribute the key, or the value is of the wrong type, the journey MUST be aborted.

### 8. Language-specific forms

How a language expresses steps, policies, hooks and adapters, such as annotations, attributes, or typed or dynamic adapters, is not part of this proposal. It is decided in each language's tech spec. For Rust, see [itinera-dev/itinera-rs#2](https://github.com/itinera-dev/itinera-rs/issues/2).

## Events

This proposal adds two engine events, whose fields and place in the catalogue are defined with the event model ([#11](https://github.com/itinera-dev/spec/issues/11)):

- an input adapter supplied a value for a step and key;
- an input adapter failed for a step and key.

## Conformance

The conformance cases for this proposal must check that:

1. a hook defined twice for the same step, or for the same workflow, is rejected at admission, before any step runs, and is reported together with other violations;
2. an input with no adapter is read from the data bag under its key;
3. an input with an adapter receives the adapter's value, and the adapter-supplied event is emitted;
4. a failing adapter, a missing input and an input of the wrong type each abort the journey, and the adapter-failed event is emitted where an adapter was involved;
5. a step hook receives exactly the value the step contributed under the requested key;
6. a step hook requesting a key the step did not contribute, or requesting the wrong type, aborts the journey;
7. the steps, order, policies and adapters of a workflow can be listed without running it.

## Alternatives considered

- **Adapters as policies.** Rejected: policies decide what happens next, adapters only move data. Mixing them would let one object change both the data and the flow, which is harder to read in the declaration.
- **Adapters declared on the step.** Rejected: each workflow knows how to adapt its own data, so adapters belong to the workflow and are associated with steps from there.
- **Adapters for data from the step.** Rejected: an adapter could supply data the step never contributed. Without one, data from the step is guaranteed to come from the step, and the hook can transform it itself.
- **Checking at admission that policies only read contributed data.** Rejected: contributions are dynamic, so a missing contribution is detected at run time and aborts the journey.
- **Policies mixing step hooks and workflow hooks.** Rejected: one type per policy keeps where it can be attached unambiguous.

## Open questions

None for this proposal. The rest of tier 1 is in [#8](https://github.com/itinera-dev/spec/issues/8) (steps), [#9](https://github.com/itinera-dev/spec/issues/9) (running a workflow), [#10](https://github.com/itinera-dev/spec/issues/10) (hooks and lifecycles), [#11](https://github.com/itinera-dev/spec/issues/11) (events) and [#12](https://github.com/itinera-dev/spec/issues/12) (the local executor).
