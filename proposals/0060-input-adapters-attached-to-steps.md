# 0060: Input adapters are attached to steps, and leave unknown inputs to the data bag

- Issue: [#60](https://github.com/itinera-dev/spec/issues/60)
- Tier: 1
- Amends: [0002](0002-foundations.md), [0042](0042-configuration-errors-before-the-journey.md)

## Summary

An input adapter is a hook of the workflow, of a new kind, `input adapter`. It is attached to steps, not to pairs of step and key. When one of its steps is built, the adapter is asked for each of the step's inputs. It returns a value, or nothing, which means "not mine": the input is then resolved the default way, from the data bag. Like every hook, it declares what it needs and receives only that: the step's name, the key, data from the workflow and the journey ID. It does not request workflow roles: a role is how a policy, reusable across workflows, sees a workflow from outside, and an adapter is part of the workflow. Adapters have names.

Amends proposal 0002 (section 6, "Data from the workflow, through input adapters") and the parts of proposal 0042 and spec 3.1, 3.4, 3.6 and 4.2 that refer to adapters associated with keys.

## Motivation

Today an adapter is associated with pairs of step and key (spec 3.4 point 1), and when it reports no value, that is final: a required input aborts the journey and an optional one is absent (4.2 point 4).

In practice an adapter adapts a step, or a few steps, to the workflow's data. It looks at which input is being asked for, supplies the ones it knows about, and leaves the rest to the data bag. Declaring every pair of step and key is repetitive, and an adapter cannot leave an input to the bag once it is associated with it. It also cannot read the data bag itself, so it cannot derive a value from data the journey already holds. For example, it cannot compute an amount from a price and a quantity.

## Design

0. **An input adapter is a hook.** It is a hook of the kind `input adapter`, defined by the workflow itself rather than by a policy, for now. It follows the rules of hooks for what it may request and how its requests are resolved (6.6 and 6.8), except where this proposal says otherwise. It acts for the workflow, never for the step, so it does not weaken the isolation of steps.
1. **Attachment.** An input adapter is declared on the workflow and attached to one or more of its steps. A step has at most one input adapter. Attaching a second one to the same step is the violation `step adapted twice`, caught when the workflow is built.
2. **Resolving an input of an adapted step.** For each input the step declares, in order, the executor calls the step's adapter. The adapter either:
   - returns a value: the input takes it, and `input_adapter_supplied` is emitted, naming the adapter;
   - returns nothing: the adapter does not supply this input, and it is resolved from the data bag as if the step had no adapter. A missing required value aborts with `required data missing`, a missing optional value is absent and emits `optional_input_absent`, and a wrong type aborts with `wrong type`;
   - throws: `input_adapter_failed` is emitted, naming the adapter, and the journey is aborted with `step could not be built` (5.10).

   A value the adapter returns that is not of the type the step declared aborts the journey with `wrong type`.
3. **What an adapter may request**, each as a declared parameter, and only what it declares:
   - the name of the step being built;
   - the key being resolved;
   - data from the workflow, by key and type, required or optional, read from the data bag and never through an adapter, with the same rules as for hooks (6.6 point 3): a required value not found aborts with `required data missing`, a wrong type aborts with `wrong type`, and an optional value not found is absent and emits `optional_input_absent`. `journey_aborted` and `optional_input_absent` name the step, the adapter and the key;
   - the journey ID (6.6 point 2).

   An adapter does not request workflow roles. Roles are how policies, which are reusable across workflows, reach the workflow from outside (6.8); an adapter is part of the workflow and its journey.

   An adapter does not receive a contributor or a means to emit events: it only supplies values. Unlike other hooks, an adapter that throws aborts the journey with `step could not be built`, not `hook failed`, as today (5.10).
4. **Order of events** for an input of an adapted step: the adapter's own `optional_input_absent` events, then `input_adapter_supplied` or `input_adapter_failed`, or, when the adapter returns nothing, the events of the default resolution.
5. **Names.** Every input adapter has a name, unique among the workflow's adapters, carried by the adapter events and shown in the listing.
6. **Listing.** For each step, the listing shows the name of its input adapter, if any, instead of the keys that have one (3.1 point 2).
7. **Admission.** An adapter attached to a step that does not exist is the violation `input adapter for unknown step`, which replaces the unnamed check for "an input adapter for a step or key that does not exist" (3.6 point 1). This also settles spec defect #59, which is closed in favour of this proposal.
8. **Unchanged:** adapters serve steps only, never hooks (6.6 point 3), and there are no adapters for data from the step.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0060-input-adapters-attached-to-steps/`, with the amended scenarios of earlier proposals tagged `@proposal-0060`.

## Alternatives considered

- **Pairs of step and key, with nothing as a final answer** (the current specification). Kept as the alternative: it lists exactly which keys are adapted, but every pair must be declared, and an adapter cannot leave an input to the bag.
- **Several adapters per step, asked in declaration order until one returns a value.** Rejected for tier 1: one adapter per step is simpler to read, and one adapter can already serve every input of a step.
- **A separate answer meaning "absent, do not read the bag".** Not included: an adapter that wants an input absent can be attached only where the bag does not hold the key. It can be added later if a case needs it.

## Open questions

- Whether, in a later tier, input adapters may be defined by policies as well as by the workflow, like other hooks.
