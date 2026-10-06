# 0058: Workflow policies are built per journey, step policies per attempt

- Issue: [#58](https://github.com/itinera-dev/spec/issues/58)
- Tier: 1
- Amends: [0002](0002-foundations.md), [0010](0010-hooks-lifecycles-and-roles.md), [0049](0049-custom-code-that-throws.md)

## Summary

Policies are built by the executor, never shared.

- **Workflow policies** are built once per journey, before it starts, since they serve the whole journey.
- **Step policies** are built for every attempt, as steps are: each attempt of a step gets new instances of the step policies attached to it.

Nothing a policy holds ever passes from one journey to another, from one step to another, or from one attempt to the next.

Amends proposal 0002 (3, "Policies") and proposal 0010, and adds rows to the table of proposal 0049.

## Motivation

The specification says policies are reusable (spec 3.3 point 3): a step policy may be attached to steps in any workflow. That is reuse of the policy's definition, but nothing says when a policy is built or how long an instance lives. Steps are built for every attempt (4.2 point 1); policies have no such rule.

It is observable. A policy that keeps anything between calls, such as a count, and contributes it from a hook produces different data bags depending on whether its instance is shared across the steps it is attached to, across attempts, across the journeys of a workflow, or not at all. If an instance were shared across journeys, one journey could change what another journey's hooks contribute, which the specification avoids everywhere else: the instance holds all journey state (5.1), and an executor keeps nothing between journeys (5.9). Building step policies with each attempt gives them the same guarantee as steps: nothing carries over from one attempt to the next (4.2 point 1).

## Design

1. **A policy's definition is reusable; its instances are not.** A policy is attached to steps or workflows by its definition. Instances are built by the executor, never shared.
2. **Workflow policies are built once per journey.** One instance of each workflow policy attached to the workflow MUST be built before the journey starts. It serves the whole journey and is discarded when the journey ends.
3. **Step policies are built for every attempt.** For each attempt of a step, one new instance of each step policy attached to that step MUST be built, right after the attempt's `attempt_started` and before its inputs are resolved. All the hooks called for that attempt use those instances, including `on step failure` after `step_given_up`. They are discarded when the attempt's hooks have returned. Nothing MUST carry over from one attempt to the next, and a step policy attached to two steps never shares an instance between them.
4. **The hooks a policy defines belong to its definition.** Which hooks a policy defines MUST be fixed by its definition, not by an instance, so `hook defined twice` is still caught at admission, before any journey (3.7), and building instances never changes it.
5. **A policy that cannot be built.** Building a policy calls custom code, so it joins the table of 5.10, with two rows:
   - **a workflow policy that throws while it is built** is a refusal: no journey starts and no event is emitted. Workflow policies are built before the journey's dispatcher is created;
   - **a step policy that throws while it is built** aborts the journey with a new abort reason, `policy could not be built`. `journey_aborted` names the step and the policy. It comes after the attempt's `attempt_started` and before any input event, so the step's status is `Aborted` and the attempt counts (spec 5.8), and no hook runs afterwards.
6. **Hooks cannot change their policy.** A hook receives everything it needs as declared parameters (6.6 and 6.8), so it has no need to keep anything in its policy instance. Where the language can make the policy instance read-only inside a hook, for example through a read-only receiver, an implementation MUST do so, and its tech spec says how. Where it cannot, points 2 and 3 still guarantee that nothing a hook changes in its policy instance outlives the journey, for a workflow policy, or the attempt, for a step policy. Conformance cannot observe this point; each language checks it with its own tests.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0058-building-policies/`, with the amended scenarios of earlier proposals tagged `@proposal-0058`.

## Alternatives considered

- **One instance per workflow or step definition, shared by every journey.** Rejected: state would leak from one journey to the next.
- **One step policy instance per step for the whole journey.** Rejected: a policy could carry state from one attempt to the next, which steps themselves are not allowed to do.
- **Building every policy before the journey starts.** Rejected for step policies: they are built with their attempt, like the step. Workflow policies serve the whole journey, so they are built first.
- **Leaving it unspecified.** Rejected: it is observable through what hooks contribute.

## Open questions

None.
