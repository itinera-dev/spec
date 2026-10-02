# 0041: Hooks read data from the workflow without input adapters

- Issue: [#41](https://github.com/itinera-dev/spec/issues/41)
- Tier: 1
- Amends: [0010: Hooks, lifecycles and workflow roles](0010-hooks-lifecycles-and-roles.md)

## Summary

Every hook, step hook or workflow hook, reads data from the workflow directly from the data bag. Input adapters serve steps only. Proposal 0010 resolved a step hook's request through the input adapter of the step it acts on.

## Motivation

A policy acts on behalf of the workflow, not of the step: it is composed into the workflow, as if it belonged to it. Input adapters exist to adapt the workflow's data to what each step expects; a policy has no such need, and reading through a step's adapter would make the same request return different values depending on the step a policy is attached to. A hook already has two clear sources: data from the step, for what the step contributed, and data from the workflow, for what the journey holds.

## Design

1. **Data from the workflow, for every hook, MUST be read from the data bag** under the requested key, by type, as required or optional. Input adapters MUST NOT be used for a hook's request.
2. **Input adapters serve steps only**, as proposal 0002 defines.
3. **Data from the step** is unchanged: exactly what the step contributed (proposal 0002).
4. **Resolving a hook's data is part of running the hook.** Everything a hook requests MUST be resolved before its body runs. If resolution fails, the hook does not run and has no `hook_called`, and the journey MUST be aborted:
   - a required value not found: `required data missing`;
   - a value of the wrong type: `wrong type`, whether the request is required or optional.

   `journey_aborted` MUST name the policy, the hook and the key.
5. **Events.** `optional_input_absent` emitted while resolving a hook's request MUST carry the policy and hook as well as the key. A hook's input events come before its `journey_*` events and its `hook_called`.
6. **This replaces** item 11 of proposal 0010, "A step hook's request MUST be resolved as an input of the step it acts on would be: through the workflow's input adapter for that step and key if there is one, otherwise from the data bag", and makes its open question "Input adapters for workflow hooks" moot.

## Events

No new events. `optional_input_absent` and `journey_aborted` name the policy and hook when they concern a hook's request.

## Conformance

The conformance cases for this proposal must check that:

1. a step hook's data from the workflow comes from the data bag even when an input adapter is declared for the step and key;
2. a hook's required data from the workflow that is missing aborts the journey with `required data missing`, naming the policy and hook, and the hook has no `hook_called`;
3. a hook's data of the wrong type aborts the journey with `wrong type`.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0041-hook-data-from-the-data-bag/`, and the amended scenario of proposal 0010 under `cases/tier-1/0010-hooks-lifecycles-and-roles/`.

## Alternatives considered

- **Resolving a step hook's request through the step's input adapter** (proposal 0010 as accepted). Rejected: a policy acts for the workflow, and adapters adapt data for steps.
- **Aborting with `hook threw` when a hook's data cannot be resolved.** Rejected: missing data and wrong types are configuration errors, and keep the same abort reasons as for steps; `journey_aborted` names the hook.

## Open questions

None.
