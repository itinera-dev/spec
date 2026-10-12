# 0095: Services provided to steps

- Issue: [#95](https://github.com/itinera-dev/spec/issues/95)
- Tier: 1
- Amends: [0002](0002-foundations.md), [0003](0003-rescue-steps.md), [0011](0011-events.md) and [0042](0042-configuration-errors-before-the-journey.md)
- Amended by [#98](https://github.com/itinera-dev/spec/issues/98): service providers may be attached to rescue steps.
- Amended by [#102](https://github.com/itinera-dev/spec/issues/102): global metadata passed to every reporter.

## Summary

A step may request **services**: objects of operations it uses to do its business, such as a payment client or a stock store, which are not values and so cannot reach it through the data bag. The workflow supplies each service through a **service provider**, part of the workflow like an input adapter, which the executor calls while a step is built. The provider declares a **scope**: an instance made once per journey and shared, or an instance made for every attempt. A provider may be attached to particular steps, replacing for them the provider of the same service that serves the others. The step requests a service by a key and a type, uses it for its attempt, and never knows where it came from, which implementation it got, or how long it lives.

Amends proposal 0002 (what a step can access, design point 1), proposal 0003 (what a rescue step receives), proposal 0011 (the event catalogue and who emits `journey_*` events) and proposal 0042 (the configuration errors caught before a journey). In the chapters it changes 1.2, 1.3, 1.5, 2.5, 2.7, 2.8, 2.9, 2.10, 3.1, 3.2, 3.5, 3.6, 3.8, 4.1, 4.2, 5.3, 5.7, 5.10, 5.11, 6.8 and 7.3 to 7.7, and adds a section on service providers to chapter 3 and one on services to chapter 4. It moves "services provided to steps" from "Later tiers" into tier 1 in ROADMAP.md.

## Motivation

A step can reach only its inputs, a contributor and a step reporter (spec 4.1). Many business units need more than data: a charge needs a payment client, a reservation needs a stock store. Those are not values, so they cannot travel through the data bag, and roles are given only to policies (6.8 point 5). Today a step author has no declared way to get one, and falls back on globals or on whatever a language's step factory happens to allow, which no conformance case can see.

Workflow authors also need to decide which implementation a step uses without the step knowing: the main payment provider on the first attempt and a backup on the next, or one stock connection shared by every step of a journey. That choice is a flow decision; it belongs to the workflow, never to the step. Today it ends up inside the step, as the fallback code VISION.md describes.

## Design

### Words

1. **Service:** an object of operations that a step uses to do its business. It is not a value: it never enters the data bag, an event, a reason's details or the result. Services carry operations for steps as roles carry operations for policies.
2. **Service provider:** part of the workflow that supplies the service of one key. It has a **name**, unique among the workflow's providers, a **key**, the **type** of what it supplies, a **scope**, and, optionally, the **steps it is attached to**.
3. **Scope:** how long an instance a provider makes is used. **Journey**: one instance, made the first time it is needed and used until the journey ends. **Attempt**: a new instance for every attempt that needs it, used for that attempt only.

### What a step requests

1. A step MAY declare services among its needs, each by a **key**, unique among the step's services, and a **type**. A step MUST be able to access only what it declares it needs: its inputs, a contributor, a step reporter, and its services.
2. A service is always required. There are no optional services in tier 1.
3. Only steps request services. Hooks of policies keep roles (6.8), and input adapters request neither.
4. The step decides nothing about a service's scope or provider: the workflow does. The step does not know which provider served it, nor whether its instance is shared.

### Which provider serves a step

1. A provider serves a step only for a service the step requests. For each service a step requests, the provider is:
   - the provider of that key attached to that step, if there is one;
   - otherwise, the provider of that key attached to no step, which serves every step of the workflow that requests the key and has no provider of its own for it.
2. A provider attached to a step that does not request its key is allowed, and is never called for that step. A provider that serves no step is checked against no step.
3. **Violations**, caught before any journey (3.7):
   - `service not provided`: a step requests a key that no provider serves for it;
   - `service requested twice`: a step requests the same key twice among its services;
   - `service provided twice`: two providers of one key are both attached to the same step, or both attached to no step, whether or not a step requests the key;
   - `duplicate provider name`: two providers of one workflow have the same name;
   - `service provider for unknown step`: a provider is attached to a step the workflow does not have;
   - `wrong service type`: the provider serving a step supplies another type than the step requests for that key;
   - `attempt request in journey scope`: a provider of journey scope requests the name of the step being built or the attempt number.
4. Implementations SHOULD report these at compile time where their language allows it.

### When a provider is called

1. **While the step is built.** Services are resolved as part of building the step (4.2): after its inputs are resolved, before the step is created, in the order the step declares its services. A step that cannot be built from its data never causes a provider to be called.
2. **Journey scope.** For each journey, the provider is called the first time an attempt of a step it serves requests its key. Every later attempt of any step it serves receives that same instance, until the journey ends. A provider attached to steps, of journey scope, makes one instance per journey shared by those steps only. A journey in which no step it serves requests its key never calls it.
3. **Attempt scope.** The provider is called for every attempt of every step it serves that requests its key, and that instance serves that attempt only. A retry is an attempt like any other.
4. **The end of a scope.** Once its scope ends, when the attempt ends or the journey ends, however it ends, the executor MUST NOT lend the instance again. How an instance is released (dropped, disposed of, closed) is each language's business, recorded in its tech spec. The executor does not cache an attempt-scoped instance, and does not share a journey-scoped instance between journeys.
5. What a provider does to make an instance (build one, return one it already holds, create one lazily) is the workflow's business. The executor guarantees only when it calls the provider and which instance it lends.

### What a provider may request

1. Each as a declared parameter, and only what it declares:
   - the journey ID;
   - data from the workflow, read from the data bag and never through an input adapter, with the rules of 6.6 point 3;
   - a means of emitting `journey_info`, `journey_warning` and `journey_error` events;
   - for a provider of attempt scope only, the name of the step being built and the attempt number. A provider of journey scope is called once, for whichever step needs it first, so it MUST NOT request them: doing so is the violation `attempt request in journey scope`.
2. A provider is part of the workflow, so it has access to the workflow's own state, as an input adapter does. It does not request roles, receives no contributor and never sees the data bag itself: it only supplies services.
3. **Resolution.** A provider's requests are resolved before it is called, in the order it declares them, with the rules of 6.6 points 3 and 4. The first that aborts the journey stops the resolution, and the provider is not called. `journey_aborted` names the step, the provider and the key of the request.
4. **Its events.** The `journey_*` events a provider emits carry the provider's name instead of a policy and hook, and the step name and attempt number of the step being built. They follow the rules of 2.9:
   - a reporter that fails on one ends the provider at its emit call and aborts the journey with `reporter failed`; neither `service_provided` nor `service_provider_failed` follows;
   - event data that is not a value, where a language can express it, aborts the journey with `not a value`, naming the step, the provider and the event kind;
   - the provider's means of emitting events is valid only during its call, like a hook's (4.6 point 7).

### What a provider returns

1. **An instance:** `service_provided` is emitted, naming the step, the attempt, the key and the provider. A step that receives an instance of journey scope made for an earlier attempt causes no event: the stream shows exactly when each instance was made.
2. **A failure**, by throwing or returning an error (5.10), or nothing at all: `service_provider_failed` is emitted, naming the step, the attempt, the key and the provider, and the journey is aborted with `step could not be built`, naming the step and the provider, with the message of the error when there is one. The step's status becomes `Aborted` and the attempt counts, as for any step that cannot be built (4.2 point 7).

### Using a service

1. A service is lent to the step for its attempt only. Implementations SHOULD make keeping a service beyond the attempt impossible to express. Where it can be expressed, what the service does afterwards is the workflow's business, and the executor never lends it again once its scope ends.
2. A service's operations are not recorded: like role calls (6.8 point 7), they are the workflow's business. A step that wants them in the stream emits `step_*` events.
3. An error from a service's operation is the step's to handle. Returned through the step's signature, or thrown out of it, it is an abnormal termination like any other (4.4).
4. A step's constructor MAY use the services it requests, since they are resolved before it is created.

### The listing and the declaration

1. A workflow declares its service providers (3.1 point 3). The workflow descriptor holds each provider's name, key, type, scope and attached steps, and a way to call it.
2. A step descriptor's needs include the keys and types of the services it requests.
3. The listing (3.1 point 2) adds, for each step, the services it requests, by key, each with the name of the provider that serves it.

### Rescue steps

A rescue step (7.1) requests services as any step does, and only providers attached to no step serve it. The workflow declares no step under its name, so a provider attached to `<step name>_rescue` is `service provider for unknown step`, as an input adapter attached to that name is, and a rescue step requesting a key that no provider attached to no step supplies is `service not provided`. A provider of attempt scope serving a rescue step receives the rescue step's name and attempt 1; for one of journey scope, the executor lends it the instance that provider made for the journey, or calls the provider for it if none was made yet.

### Execution modes

A provider is synchronous or asynchronous, under the executor's execution modes, like a hook (5.11). A provider in a mode the executor does not accept is `mode not accepted`.

### Rules a language may make impossible to express (3.8)

1. Item 3 also covers a provider in an execution mode the executor does not accept.
2. Item 5 also covers a service kept and used after its attempt, and a provider's means of emitting events used after its call. There is no conformance case for a kept service: what it does is the workflow's business.
3. A new item 8, after the rescue step's item 7, under the tag `@journey-scope-request`: a provider of journey scope requesting the step's name or the attempt number.
4. A new item 9, under the tag `@no-instance`: a provider returning nothing, neither an instance nor an error.

## Events

Two new facts, both about a step, carrying the step name and the attempt number:

| Event | When | Also carries |
|---|---|---|
| `service_provided` | a provider returned an instance for a step's service | key, provider |
| `service_provider_failed` | a provider failed, or returned nothing | key, provider |

`journey_aborted` gains the provider, for an abort because a provider failed, returned nothing, or emitted data that is not a value, and the provider and the key of the request, for an abort while its requests were resolved.

**Order** (2.8, point 2): after the input events, and before the step's own `step_*` events, the service events, in the order the step's services are resolved. For each service whose provider is called: the provider's own `optional_input_absent` events, then the `journey_*` events it emits, then `service_provided` or `service_provider_failed`. Nothing for a service of journey scope already made.

`journey_info`, `journey_warning` and `journey_error` may now come from a provider as well as from a hook (2.9 point 2).

## Conformance

Cases in `cases/tier-1/0095-services-provided-to-steps/`:

1. A journey-scoped service requested by two steps is provided once: one `service_provided`, at the first step.
2. An attempt-scoped service is provided for each attempt, retries included, and for each step that requests it: one `service_provided` per attempt; a journey-scoped one serves every attempt of a retried step with one instance.
3. A provider attached to two steps serves them, and the provider of the same key attached to no step serves the others, for all but two steps.
4. A journey-scoped provider attached to steps is shared by those steps only.
5. An attached provider needs no provider of the same key for the other steps; one attached to a step that does not request its key is never called; and an unattached provider of another type that serves no step is not checked against one.
6. An attempt-scoped provider receives the step's name and the number of each attempt, and a provider of either scope the journey ID.
7. An attempt-scoped provider that reads data from the workflow lends a different instance after a retry hook contributed that data.
8. A provider is never called in a journey in which no step it serves runs, one whose step comes after a step that fails, nor for a step that does not request its service.
9. A provider's required data missing, or of the wrong type: `journey_aborted` with `required data missing` or `wrong type`, naming the step, the provider and the key; the provider is not called and the step does not run.
10. A provider's optional data absent: `optional_input_absent` naming the provider, then `service_provided`.
11. A provider that fails: `service_provider_failed`, then `journey_aborted` with `step could not be built`, naming the step and the provider, with the error's message; the step is `Aborted`, also when it fails on a retry, where the attempt counts.
12. Tagged `@no-instance`: a provider that returns nothing aborts the journey in the same way, with no error.
13. A provider's `journey_*` events, in order before `service_provided`, carrying the provider's name; a reporter that fails on one aborts the journey with `reporter failed`, and no service event follows; tagged `@non-value`, event data that is not a value aborts it with `not a value`, naming the provider.
14. Services are resolved after inputs, in the order the step declares them: a step whose required input is missing aborts with `required data missing` and its provider is not called; a step's input events come before its service events.
15. Admission, each refused with no event: `service not provided` (also when the only provider is attached to other steps), `service requested twice`, `service provided twice` (both forms, and when no step requests the key), `duplicate provider name`, `service provider for unknown step`, `wrong service type`, every violation reported at once, `attempt request in journey scope` (tagged `@journey-scope-request`), and an asynchronous provider under a synchronous-only executor, `mode not accepted` (tagged `@mode-not-accepted`).
16. The listing shows each step's services and the provider serving each.
17. An error from a service's operation, returned through the step, is an abnormal termination.
18. A rescue step is served by the provider attached to no step: an attempt-scoped one makes it an instance for its attempt 1, and a journey-scoped one lends it the journey's instance; a provider attached to `<step>_rescue` is `service provider for unknown step`.

## Alternatives considered

- **The step picks the scope.** Rejected: the step would decide how something it must not know about is made, every provider would have to support both scopes, and the provider for a retry could not be chosen by the workflow. Workflows know steps; steps do not know workflows.
- **The workflow's own fields handed out by reference, scope only a convention.** Rejected: the specification could not promise when an instance is made or shared, and conformance could not check it.
- **Providers as hooks of policies.** Rejected: a policy reaches the workflow only through roles, so the workflow would need a role just to hand its clients to the policy. A provider is called while a step is built and returns an object, as an input adapter does, so it belongs with input adapters, as part of the workflow.
- **Two keys for two scopes** (a shared and a fresh variant of one service). Rejected: the step would choose by key, which is the step picking the scope.
- **Services for hooks.** Rejected for tier 1: hooks have roles for operations.
- **Optional services.** Left out: nothing needs a step that runs with or without a service yet.

## Open questions

None for tier 1. A release hook for instances (closing a connection when a scope ends) is left to each language; a later proposal may add one if languages need a common behaviour.
