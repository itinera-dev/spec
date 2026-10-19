# 0116: Duplicate adapter names, and a step named twice in one list

- Issue: [#116](https://github.com/itinera-dev/spec/issues/116)
- Tier: 1
- Amends: [0060](0060-input-adapters-attached-to-steps.md) and [0095](0095-services-provided-to-steps.md)

## Summary

Two input adapters of one workflow with the same name are the new violation `duplicate adapter name`, as two service providers with the same name are already `duplicate provider name`. An input adapter and a service provider may have the same name. An input adapter or a service provider whose list of steps names one step more than once is attached to that step more than once, which is the existing violation `step adapted twice` for an adapter and `service provided twice` for a provider. No event, abort reason, status or lifecycle is added.

Amends proposal 0060, which made adapter names unique among a workflow's adapters without naming a violation for two of the same name, and proposal 0095, whose `service provided twice` did not say what one provider naming a step twice is. In the chapters it changes 3.4, 3.6 and 3.9.

## Motivation

1. **Adapter names.** An input adapter's name is how the event stream and the listing tell adapters apart: `input_adapter_supplied`, `input_adapter_failed`, `optional_input_absent` and `journey_aborted` name the adapter, and the listing shows each step's adapter by name. Two adapters of one workflow with the same name cannot be told apart in either. Chapter 3, 3.4 point 1 forbade them, but 3.6, which names every configuration error, named no violation for them. An implementation that refused such a workflow had no name to report, and a conformance case could not check it. Service providers have the same rule and a violation for it, `duplicate provider name` (3.9 point 3.2).
2. **A step named twice in one list.** An input adapter and a service provider are each attached to a list of steps (3.4 point 1, 3.9 point 1). The specification said what two adapters attached to one step are, `step adapted twice`, and what two providers of one key attached to one step are, `service provided twice`, but not what one adapter or provider that names the same step twice in its list is. It could be read as harmless, as an error with no name, or as one of the existing violations, and two implementations could differ.

## Design

1. **`duplicate adapter name`.** Two input adapters of one workflow with the same name MUST be refused with the violation `duplicate adapter name`, whether or not they are attached to the same steps. It is an error in how the workflow is put together, reported when the workflow is built (3.7 point 1), with every other violation found (3.7 point 3).
2. **Adapter and provider names are separate.** An input adapter's name is unique among the workflow's input adapters, and a service provider's among its service providers (3.4 point 1, 3.9 point 1). An input adapter and a service provider MAY have the same name: events and the listing name each in its own place.
3. **One adapter naming a step twice.** An input adapter whose list of steps names one step more than once is attached to that step more than once, and MUST be refused with the violation `step adapted twice`, as when two adapters are attached to the step. The same holds for a rescue step's name, `<step name>_rescue` (3.4 point 2).
4. **One provider naming a step twice.** A service provider whose list of steps names one step more than once is attached to that step more than once, and MUST be refused with the violation `service provided twice`, as when two providers of one key are attached to the step. The same holds for `<step name>_rescue` (3.9 point 7.2).
5. **The meanings in 3.6 point 2.** The table of violations gains `duplicate adapter name`: two input adapters of one workflow have the same name. `step adapted twice` becomes: two input adapters are attached to the same step, or one input adapter is attached to it more than once. `service provided twice` becomes: two providers of one key are both attached to the same step, or both attached to no step, or one provider is attached to the same step more than once.
6. **The list in 3.6 point 1** names two input adapters with the same name among the errors in how a workflow is put together, and, beside two adapters or two providers of one key attached to one step, one adapter or provider attached to a step more than once.

Nothing else changes: the listing, the events and how steps are built are as before.

## Events

None added or changed.

## Conformance

The cases live with those of the proposals this one amends, tagged `@proposal-0116`, in itinera-dev/conformance: `cases/tier-1/0060-input-adapters-attached-to-steps/input-adapters.feature`, `cases/tier-1/0003-rescue-steps/input-adapters.feature`, `cases/tier-1/0095-services-provided-to-steps/admission.feature` and `cases/tier-1/0095-services-provided-to-steps/rescue-steps.feature`. They check that:

1. two input adapters with the same name, attached to different steps, are refused with `duplicate adapter name`, and no event is emitted;
2. an input adapter and a service provider with the same name are accepted;
3. an input adapter that names one step twice in its list is refused with `step adapted twice`, and no event is emitted;
4. an input adapter that names one rescue step twice in its list is refused with `step adapted twice`, and no event is emitted;
5. a service provider that names one step twice in its list is refused with `service provided twice`, and no event is emitted;
6. a service provider that names one rescue step twice in its list is refused with `service provided twice`, and no event is emitted.

STEPS.md gains the violation `duplicate adapter name` and the new meanings of `step adapted twice` and `service provided twice`, its sentence declaring an input adapter says that a second declaration of the same name declares a second adapter, and its sentences attaching an adapter or a provider to steps say that a step named twice is attached twice. No existing case changes.

## Alternatives considered

1. **A step named twice in one list is one attachment.** The repetition would be ignored. Rejected: a step written twice is most likely a slip for another step, which then silently gets no adapter or provider, and a declaration should show each attachment once.
2. **A new violation for a step named twice in one list,** such as `step listed twice`. Rejected: the step has two attachments of one kind either way, and the existing violations already name that.
3. **Leave adapter names unchecked.** Rejected: events and the listing name adapters, and two adapters with the same name could not be told apart in either.
4. **One set of names for adapters and providers together.** Not proposed: they are named in different fields of events and columns of the listing, so they cannot be confused, and nothing in the specification relies on it.

## Open questions

None.
