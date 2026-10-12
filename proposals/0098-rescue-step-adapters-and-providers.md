# 0098: Input adapters and service providers for rescue steps

- Issue: [#98](https://github.com/itinera-dev/spec/issues/98)
- Tier: 1
- Amends: [0003](0003-rescue-steps.md), [0060](0060-input-adapters-attached-to-steps.md) and [0095](0095-services-provided-to-steps.md)

## Summary

A rescue step may have its own input adapter and its own service providers. They are attached to its name, `<step name>_rescue`, in the same way as to a declared step. The rescue step gets only what is attached to that name, and nothing attached to the step it rescues. The listing shows the rescue step's adapter and services in a rescue section of its step's entry.

Amends proposal 0003 (what a rescue step is declared with, what it receives and what the listing shows), proposal 0060 (which names an input adapter may be attached to) and proposal 0095 (which providers serve a rescue step). In the chapters it changes 3.1 point 2, 3.2, 3.4 point 2, 3.6, 3.9 points 3 and 7, 3.10, 7.1 points 1, 2 and 5, 7.3 points 1 and 2, 7.5 point 5 and 7.7.

## Motivation

Today a rescue step reads its inputs only from the data bag, and only providers attached to no step serve it (spec 7.3 point 1, 3.9 point 7). An input adapter or a provider attached to `<step name>_rescue` is refused as attached to an unknown step.

That leaves a rescue step without data its step had. When an input adapter supplies an input of the rescued step from data of the workflow, the value goes to that step and never enters the data bag. The rescue step cannot request it, and no adapter can supply it, so the rescue step has no declared way to get the data it needs to repair the failure.

It also leaves the workflow unable to choose a service for the rescue. A rescue step often does a different kind of work from its step, such as refreshing a token or switching to a backup, and so it may need a different implementation of a service than the other steps get. Today it can only get the implementation that serves every step.

## Design

### Attaching to a rescue step

1. **A rescue step's name may be used for attachment.** Where an input adapter or a service provider is attached to steps, the name of a rescue step, `<step name>_rescue`, MAY be named as if it were a step. The rescue step stays undeclared: it still has no step descriptor, no step policies, no retry budget and no `abnormal termination retriable` setting.
2. **Only for a step that has a rescue step.** Attaching to `<step name>_rescue` when the step has no rescue step is the violation `input adapter for unknown step` for an adapter, and `service provider for unknown step` for a provider, as for any name the workflow does not have.
3. **Nothing is inherited.** An adapter or provider attached to a step serves that step only, never its rescue step. A rescue step gets only what is attached to its own name and, for services, the providers attached to no step.

### The rescue step's input adapter

1. A rescue step has at most one input adapter. Two adapters attached to `<step name>_rescue` are the violation `step adapted twice`. One adapter MAY be attached both to a step and to its rescue step, as to any two steps.
2. When the rescue step is built, its inputs are resolved as for an adapted step (3.4 and 4.2). For each input it declares, in order, its adapter is asked. A value the adapter returns becomes the input. When the adapter returns nothing, the input is resolved from the data bag. The same events and aborts apply, naming the rescue step: `input_adapter_supplied`, `input_adapter_failed` with `step could not be built`, `wrong type`, `required data missing` and `optional_input_absent`.
3. The adapter MAY request what any adapter may. When it requests the name of the step being built, it receives `<step name>_rescue`.
4. Data that the rescue step requests from the step it rescues is never asked of the adapter: the failed attempt's contributions, the failure's reason and the error (7.3 point 2). Adapters serve inputs only.

### The rescue step's services

1. For each service a rescue step requests, the provider is the one of that key attached to `<step name>_rescue`, if there is one. Otherwise it is the provider of that key attached to no step. This is the rule for any step (3.9 point 2).
2. The violations of 3.9 apply to the rescue step as to a step. Two providers of one key attached to `<step name>_rescue` are `service provided twice`. A key that neither kind of provider supplies is `service not provided`. A provider of another type than the rescue step requests is `wrong service type`.
3. A provider of journey scope attached to a step and to its rescue step makes one instance per journey, which they share. A provider of journey scope attached to `<step name>_rescue` alone makes one for the rescue step.
4. A provider of attempt scope serving a rescue step receives the rescue step's name and attempt 1, as today.

### The listing

1. A step that has a rescue step shows, in its entry, a **rescue section**. That section holds the rescue step's name, the name of its input adapter if it has one, and the services it requests, by key in the order it declares them, each with the name of the provider that serves it.
2. The rescue section has no position, since the rescue step runs only when its step fails, and no policies, since a rescue step has none.
3. A step without a rescue step has no rescue section.

### Unchanged

- A rescue step's name remains reserved: a declared step named `<step name>_rescue` is still `duplicate step name`.
- Hooks still get nothing from input adapters, and only steps and rescue steps request services.

## Events

No new events. The input adapter and service provider events name the rescue step when they are emitted for it.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), tagged `@proposal-0098`, next to the proposals they amend: in `cases/tier-1/0003-rescue-steps/declaring.feature`, in a new file `cases/tier-1/0003-rescue-steps/input-adapters.feature` for the new adapter cases, and in `cases/tier-1/0095-services-provided-to-steps/rescue-steps.feature`.

These cases change:

1. In `0003-rescue-steps/declaring.feature`, the scenario "An input adapter cannot be attached to a rescue step" is replaced by one in which an adapter attached to `<step name>_rescue` supplies the rescue step's input.
2. In `0095-services-provided-to-steps/rescue-steps.feature`, the scenario "A provider attached to a rescue step's name is refused" is replaced by one in which a provider attached to `<step name>_rescue` serves the rescue step instead of the provider attached to no step.

These cases are new:

3. The listing shows the rescue section, with the rescue step's adapter and services, next to the scenario "The listing shows each step's rescue step", which stays as it is.
4. An adapter attached to `<step name>_rescue` that returns nothing leaves the input to the data bag.
5. An adapter attached to `<step name>_rescue` receives the rescue step's name.
6. An adapter attached to a step is not asked for its rescue step's inputs.
7. Data from the rescued step is never asked of the rescue step's adapter.
8. Two adapters attached to `<step name>_rescue` are `step adapted twice`.
9. An adapter attached to `<step name>_rescue` when the step has no rescue step is `input adapter for unknown step`.
10. A provider attached to `<step name>_rescue` when the step has no rescue step is `service provider for unknown step`.
11. Two providers of one key attached to `<step name>_rescue` are `service provided twice`.
12. A provider of journey scope attached to a step and to its rescue step lends them the same instance.

The existing scenario "A rescue step is not served by a provider attached to the step it rescues" stays as it is.

## Alternatives considered

- **The rescue step inherits its step's attachments unless something is attached to its own name.** Rejected by the maintainer: what a rescue step receives would be implied rather than declared, and a rescue step often needs something different from its step.
- **Listing rescue steps in a separate section after the steps.** Rejected: the rescue step belongs to its step and runs only in its place, so it is listed in its step's entry.
- **Keeping only the rescue step's name in the listing.** Rejected: an adapter or provider attached to a rescue step would then appear nowhere in the listing.

## Open questions

None.
