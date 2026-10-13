# 4. Steps

Specification 0.1. Written from proposals [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md) as amended by [0024](../proposals/0024-abnormal-termination-retriable.md), and [0027](../proposals/0027-received-data-read-only.md), with the amendments [0054](../proposals/0054-how-custom-code-fails.md), [0055](../proposals/0055-reporter-fails-while-custom-code-runs.md), [0056](../proposals/0056-data-bag-values.md), [0057](../proposals/0057-handles-valid-during-their-attempt.md), [0060](../proposals/0060-input-adapters-attached-to-steps.md), [0064](../proposals/0064-event-data-values.md), [0091](../proposals/0091-adapters-read-the-data-bag.md), [0095](../proposals/0095-services-provided-to-steps.md), [0102](../proposals/0102-global-metadata.md) and [0104](../proposals/0104-service-implementation-on-events.md), and from proposal [0003](../proposals/0003-rescue-steps.md).

What a step is, how it is built and run, the outcomes it reports, and what happens to the data it contributes. When a step runs, and what follows its outcome, are defined in chapters 5 and 6.

## 4.1 What a step can access

1. A step MUST be able to access only what it declares it needs:
   - its **inputs**: data from the workflow;
   - a **contributor**, for the data it contributes back;
   - a **step reporter**, for its own events and for setting global metadata (chapter 2, 2.9 and 2.11);
   - its **services**: objects of operations the workflow provides for its business (4.9);
   - for a rescue step only, **data from the step it rescues** (chapter 7, 7.3).
2. A step MUST NOT have access to the workflow, the data bag, the executor, the input adapters, the service providers, other steps, or anything else that runs it.
3. A step does not know its own name: an implementation MUST NOT provide a step with its name. A step that needs a label receives it as an ordinary input.

## 4.2 Building a step

1. **Built on demand, once per attempt.** An implementation MUST build a fresh step for every attempt. Nothing MUST carry over from one attempt to the next, except an instance of journey scope that a service provider made (4.9).
2. **Declared inputs.** A step declares each input by a **key**, local to the step, a **type**, and whether it is **required** or **optional**.
3. **Resolving an input.** Resolving inputs is part of building the step: every input is resolved, outside the step, before the step is created, so a built step has all its inputs. For each input:
   1. **If the step has an input adapter** (chapter 3, 3.4), the executor asks it, for each input in the order the step declares them. Its own requests for data from the workflow are resolved first, in the order the adapter declares them, with the rules of chapter 6, 6.6 point 3; the first that aborts the journey stops the resolution, the adapter is not called, and `journey_aborted` names the step, the adapter and the key. The adapter then either:
      - **returns a value**: the input takes it, and `input_adapter_supplied` is emitted, naming the adapter;
      - **returns a value of the wrong type**: `input_adapter_supplied` is emitted, naming the adapter, and the journey is aborted with `wrong type`, naming the step, the adapter and the key;
      - **returns nothing**: the adapter does not supply this input, which is resolved from the data bag, as in point 2;
      - **fails, or returns something that is not a value**: `input_adapter_failed` is emitted, naming the adapter, and the journey is aborted with `step could not be built`.
   2. **Otherwise**, and for an input the adapter returned nothing for, the value MUST be read from the data bag under the key.
4. **No value.** When the data bag has no value under the key:
   - for a **required** input, the journey MUST be aborted with the abort reason `required data missing`, naming the step and the key. When the step has an input adapter that returned nothing for this input, the adapter is not named: it supplied nothing;
   - for an **optional** input, the step MUST be built with that input absent, and `optional_input_absent` is emitted.
5. **Wrong type.** A value of the wrong type MUST abort the journey with the abort reason `wrong type`, whether the input is required or optional, naming the step and the key, and the adapter when the adapter supplied the value.
6. **Other build failures.** An input adapter that fails, a service provider that fails (4.9), a constructor that fails, or any other reason a step cannot be built MUST abort the journey with the abort reason `step could not be built`.
7. **Building fails: the journey is aborted.** A step that cannot be built makes the run unsafe. Its status becomes `Aborted`, and it MUST NOT enter the retry loop.
8. **An input adapter's access to the data bag.** An input adapter MAY request read access to the data bag, as one of its declared requests (chapter 3, 3.4 point 3), resolved in the order it declares them.
   1. **What it shows.** For the call it is given to, it lets the adapter look up any key and read its value as a type the adapter names. It shows the data bag the step is being built from: everything committed before this attempt, including the contributions of the hooks of earlier attempts (chapter 6, 6.7), and never the contributions of an earlier attempt that did not succeed (4.6).
   2. **Read-only.** Nothing the adapter does through the access, or with what it reads, may change the data bag. Implementations SHOULD make writing through the access impossible to express; where a write can be expressed, it MUST NOT reach the data bag, nor change what any step, hook or adapter receives afterwards. Every value read is the adapter's own copy (4.7).
   3. **Reads are the adapter's own.** A read through the access is not a request resolved by the executor: a key not in the data bag is reported to the adapter as absent, and a value of another type than the one asked for is reported as such. Neither emits an event or aborts the journey. The adapter then returns a value, returns nothing, or fails, as point 3.1 says.
   4. **Valid only during the call.** Used after the call it was given to, the access MUST NOT reveal anything of the data bag: every lookup reports the key absent. It does not fail, since a later use happens inside another call of the same journey.
9. **Services.** After its inputs are resolved, and before the step is created, its services are resolved, in the order the step declares them, as 4.9 defines. A step that cannot be built from its data never causes a service provider to be called.

## 4.3 Running a step: outcomes

1. **An outcome states what happened, never what to do next.** A step that runs to the end MUST report exactly one of:
   - **Success**: the step completed;
   - **Failure**: the step could not complete;
   - **Skipped**: the step decided, from what it knows of its domain, not to do its work.
2. **The retriable flag.** Every failure carries a **retriable** flag, false unless the step sets it:
   - **not retriable**: trying again would not help;
   - **retriable**: trying again might help.

   The flag is the step's suggestion, not a decision: whether a retriable failure is tried again depends on the step's retry budget and the hooks (chapters 5 and 6).
3. **Reasons.** A failure MUST carry a **reason**: a **code** (a string), an optional **message**, and optional **details**, which MUST be a value (4.6) and are captured when the reason is made; the executor never serializes them. A skip MAY carry a reason of the same shape. A reason whose details are not a value aborts the journey with `not a value`.

## 4.4 Abnormal termination

1. If a built step lets a recoverable failure escape while running, an exception or an error returned through its signature, the attempt MUST end with an **abnormal termination**, and `step_abnormal_termination` is emitted. The error is all the hooks receive about it. An unrecoverable failure, such as a panic in Rust, is outside the model (chapter 5, 5.10).
2. The executor's own signal that ends a step at an emit or contribute call (chapter 2, 2.9 point 5) is never an abnormal termination.
3. Whether it is retried depends on the step descriptor's `abnormal termination retriable` setting (chapter 5). By default it is not: an error the step did not anticipate may already have had side effects.
4. An abnormal termination MUST NOT abort the journey. It always ends as a failure of the step, at once or after its retries.
5. An abnormal termination carries no contributions.

## 4.5 Skipped

1. **A skipped step is out of the flow.** No hook MUST be called for it, and no lifecycle can follow it: the executor moves to the next step.
2. **A skipped step has no effect on data.** Everything it contributed MUST be discarded, and `contributions_discarded` is emitted. A later step that requires a key only the skipped step contributed is aborted with `required data missing`; one that requests it as optional is built with it absent.
3. **A skipped step is final.** Its status is `Skipped`, and it MUST NOT run again in the journey.
4. **Only the step decides to skip.** Hooks cannot skip steps.
5. **A rescue step MUST NOT skip.** A rescue step that reports Skipped aborts the journey with `rescue skipped` (chapter 7, 7.4 point 4).

## 4.6 Contributions

1. **Contributions are dynamic.** The contributor accepts a key and a value; keys are not declared in advance. A value of the wrong type is detected when it is read.
2. **Everything in the data bag is a value**: data that can be serialized (chapter 1, 1.3). Functions, closures, connections, handles and other objects that are not data never enter it. Tier 1 serializes nothing; the rule says what may be stored, so that a later executor can serialize it. Implementations SHOULD make storing a non-value impossible to express (chapter 3, 3.8). Where it can be expressed, contributing something that is not a value is illegal: the journey is aborted with `not a value`, naming the step, or the policy and hook, and the key, and the contribute call ends the step or hook with the executor's signal, as for a reporter that fails (chapter 2, 2.9 point 5).
3. **The same key twice in one attempt:** the last value MUST win.
4. **Captured when made.** A value passed to a contributor MUST be captured at that moment: changing the object afterwards MUST NOT change what was contributed.
5. **Visible to hooks, committed only on success.** A step's contributions MUST be visible to the hooks of the attempt that made them, whatever its outcome, and MUST be committed to the data bag only when the attempt succeeds. The contributions of a failed attempt, retriable or not, MUST NOT reach later steps or the result.
6. **Overwriting.** A committed contribution MAY replace a key already in the data bag, including one from the initial data. The last committed value wins, and `data_overwritten` is emitted.
7. **Handles are valid only during their attempt.** During its attempt, a step contributes and emits freely, in any order. Its contributor and step reporter are made by the executor for that attempt and closed when the attempt ends; closing is idempotent. A call on a closed handle runs no custom code and has no effect on any journey: nothing is recorded, shown to hooks or committed, no event is delivered and no global metadata is set. The call SHOULD fail where it is made, so the misuse is visible, but that failure MUST NOT change any journey. Implementations SHOULD make such a call impossible to express (chapter 3, 3.8). The same holds for a hook's contributor and its means of emitting events, until the hook returns.

## 4.7 Received data is read-only

1. Every value a step receives as an input is read-only for it.
2. Changing a received value, where the language allows it at all, MUST NOT change the data bag, the contributions of any step or hook, or what any other step or hook receives.
3. The only way to change the data bag is a contributor, so every change is visible in the events.
4. How this is guaranteed is each language's choice, recorded in its tech spec: for example through ownership and borrowing, or by giving each receiver its own copy or an immutable view.

## 4.8 Time

1. **The executor never interrupts custom code that does nothing illegal.** It does not interrupt a step's execution at any stage (building it, running it, its events, or the hooks after it), and it does not cancel futures, stop threads or impose time limits on a step while it runs.
2. **When custom code does something illegal** and the executor learns of it, the executor aborts the journey. Where an executor-owned signal, given at a call the step or hook itself made, is the only way to stop it, the executor ends it there (chapter 2, 2.9 point 5); whatever the code does afterwards is ignored, and the abort stands.
3. A step handles its own time limits, for example a timeout on a call it makes, and reports a failure, retriable or not, when it runs out of time.

## 4.9 Services

1. **Requesting a service.** A step MAY declare services among its needs, each by a **key**, unique among the step's services (a key requested twice is the violation `service requested twice`, chapter 3, 3.9), and a **type**. A service is always required. Which provider serves each service is defined in chapter 3, 3.9; a step requesting a service no provider serves is refused at admission.
2. **The step knows nothing of where a service comes from.** It does not know which provider served it, which implementation it received, whether its instance is shared, or how long it lives. It decides nothing about any of these: the workflow does.
3. **Only steps request services.** Hooks of policies request roles (chapter 6, 6.8), and input adapters request neither.
4. **When a provider is called.** Services are resolved while the step is built (4.2 point 9):
   1. **Journey scope.** For each journey, the provider is called the first time an attempt of a step it serves requests its key. Every later attempt of any step it serves receives that same instance, until the journey ends. A provider of journey scope attached to steps makes one instance per journey, shared by those steps only. A journey in which no step it serves requests its key never calls it.
   2. **Attempt scope.** The provider is called for every attempt of every step it serves that requests its key, and that instance serves that attempt only. A retry is an attempt like any other.
   3. **The end of a scope.** Once its scope ends, when the attempt ends or when the journey ends, however it ends, the executor MUST NOT lend the instance again. It never shares an instance of journey scope between journeys, and never keeps an instance of attempt scope for another attempt. How an instance is released is each language's choice, recorded in its tech spec.
   4. What a provider does to make an instance, such as building one, returning one it already holds or creating one lazily, is the workflow's business. The executor guarantees only when it calls the provider and which instance it lends.
5. **A provider's requests** (chapter 3, 3.9 point 4) are resolved before it is called, in the order it declares them, with the rules of chapter 6, 6.6 points 3 and 4. The first that aborts the journey stops the resolution, and the provider is not called; `journey_aborted` names the step, the provider and the key of the request. An optional request with no value is answered absent, and `optional_input_absent` is emitted, naming the provider.
6. **What a provider returns.**
   1. **An instance:** `service_provided` is emitted, naming the step, the attempt, the key, the provider and the implementation it made, and carrying the provider's data, if it supplied any (points 6.3 and 6.4). A step that receives an instance of journey scope made for an earlier attempt causes no event, so the event stream shows exactly when each instance was made, and its implementation and data appear once, on the `service_provided` of the call that made it.
   2. **A failure,** by throwing or returning an error (chapter 5, 5.10), **or nothing at all:** `service_provider_failed` is emitted, naming the step, the attempt, the key and the provider, but no implementation and no data, since no instance was made, and the journey is aborted with `step could not be built`, naming the step and the provider, with the message of the error when there is one. The step's status becomes `Aborted`, and the attempt counts (4.2 point 7).
   3. **The implementation.** `service_provided` MUST carry an **implementation**: a string naming the instance the provider returned. It SHOULD be the name of the instance's concrete type, as the language reports it at run time, and not the name of the service type the step requested, so that two providers of one key making instances of different types give different implementations. Where a language cannot tell an instance's concrete type, such as an object with no class of its own, the implementation is the name of the type the provider declares it supplies, and the language's tech spec records when that happens. The format of the name is the language's own, and MAY change between versions of the language or its toolchain: reporters SHOULD NOT depend on it. A provider cannot replace the implementation with a name of its own.
   4. **The provider's data.** With an instance, a provider MAY supply **data**, which `service_provided` carries; when it supplies none, the event carries no data. The data MUST be a value (4.6), as the data of a `journey_*` event (chapter 2, 2.9 point 3): the event carries its own copy, captured when the provider returns, and the executor MUST NOT serialize it. It is for reporters only: it never enters the data bag or the result, the step never sees it, and no step, hook, input adapter or service provider can read it. Data that is not a value is illegal: where a language can express it, the journey is aborted with `not a value`, naming the step, the provider and the event kind `service_provided`; neither `service_provided` nor `service_provider_failed` is emitted, the step is not given the instance, and its status becomes `Aborted` (4.2 point 7). An instance of journey scope made in that call is never lent.
7. **A provider's events.** A provider MAY emit `journey_info`, `journey_warning` and `journey_error`, which carry its name and the step name and attempt number of the step being built (chapter 2, 2.9), and MAY set global metadata through the same means (chapter 2, 2.11).
   1. **A reporter that fails** on one, or on an entry of global metadata the provider set, ends the provider at its call, and the journey is aborted with `reporter failed`. Neither `service_provided` nor `service_provider_failed` follows, and whatever the provider returns afterwards is ignored.
   2. **Event data that is not a value**, where a language can express it, aborts the journey with `not a value`, naming the step, the provider and the event kind, and ends the provider at its emit call in the same way.
   3. **Valid only during the call.** The provider's means of emitting events is made for one call and closed when the call returns, with the rules of 4.6 point 7 for a closed handle.
8. **Using a service.**
   1. A service is lent to the step for its attempt only. Implementations SHOULD make keeping a service beyond the attempt impossible to express (chapter 3, 3.8). Where it can be expressed, what the service does afterwards is the workflow's business, and the executor never lends it again once its scope ends.
   2. A step's constructor MAY use the services the step requests, since they are resolved before it is created.
   3. A service's operations are not recorded: like role calls, they are the workflow's business. A step that wants them in the event stream emits `step_*` events.
   4. An error from a service's operation is the step's to handle. Returned through the step's signature, or thrown out of it, it is an abnormal termination like any other (4.4).
9. **A service is not a value.** It never enters the data bag, an event, a reason's details or the result (4.6).

## 4.10 Checked by

In the conformance repository: [`0002-foundations/data-from-the-workflow.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/data-from-the-workflow.feature), [`0008-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0008-steps) (building, outcomes, skipped and contributions), [`0024-abnormal-termination-retriable/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0024-abnormal-termination-retriable) [`0027-received-data-read-only/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0027-received-data-read-only), [`0056-data-bag-values/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0056-data-bag-values), [`0057-handles-valid-during-their-attempt/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0057-handles-valid-during-their-attempt), [`0060-input-adapters-attached-to-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0060-input-adapters-attached-to-steps) and [`0095-services-provided-to-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0095-services-provided-to-steps).
