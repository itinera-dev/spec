# 4. Steps

Specification 0.1. Written from proposals [0002](../proposals/0002-foundations.md), [0008](../proposals/0008-steps.md) as amended by [0024](../proposals/0024-abnormal-termination-retriable.md), and [0027](../proposals/0027-received-data-read-only.md), with the amendments [0054](../proposals/0054-how-custom-code-fails.md), [0055](../proposals/0055-reporter-fails-while-custom-code-runs.md), [0056](../proposals/0056-data-bag-values.md), [0057](../proposals/0057-handles-valid-during-their-attempt.md), [0060](../proposals/0060-input-adapters-attached-to-steps.md) and [0064](../proposals/0064-event-data-values.md).

What a step is, how it is built and run, the outcomes it reports, and what happens to the data it contributes. When a step runs, and what follows its outcome, are defined in chapters 5 and 6.

## 4.1 What a step can access

1. A step MUST be able to access only what it declares it needs:
   - its **inputs**: data from the workflow;
   - a **contributor**, for the data it contributes back;
   - a **step reporter**, for its own events (chapter 2).
2. A step MUST NOT have access to the workflow, the data bag, the executor, the input adapters, other steps, or anything else that runs it.
3. A step does not know its own name: an implementation MUST NOT provide a step with its name. A step that needs a label receives it as an ordinary input.

## 4.2 Building a step

1. **Built on demand, once per attempt.** An implementation MUST build a fresh step for every attempt. Nothing MUST carry over from one attempt to the next.
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
6. **Other build failures.** An input adapter that fails, a constructor that fails, or any other reason a step cannot be built MUST abort the journey with the abort reason `step could not be built`.
7. **Building fails: the journey is aborted.** A step that cannot be built makes the run unsafe. Its status becomes `Aborted`, and it MUST NOT enter the retry loop.

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

## 4.6 Contributions

1. **Contributions are dynamic.** The contributor accepts a key and a value; keys are not declared in advance. A value of the wrong type is detected when it is read.
2. **Everything in the data bag is a value**: data that can be serialized (chapter 1, 1.3). Functions, closures, connections, handles and other objects that are not data never enter it. Tier 1 serializes nothing; the rule says what may be stored, so that a later executor can serialize it. Implementations SHOULD make storing a non-value impossible to express (chapter 3, 3.8). Where it can be expressed, contributing something that is not a value is illegal: the journey is aborted with `not a value`, naming the step, or the policy and hook, and the key, and the contribute call ends the step or hook with the executor's signal, as for a reporter that fails (chapter 2, 2.9 point 5).
3. **The same key twice in one attempt:** the last value MUST win.
4. **Captured when made.** A value passed to a contributor MUST be captured at that moment: changing the object afterwards MUST NOT change what was contributed.
5. **Visible to hooks, committed only on success.** A step's contributions MUST be visible to the hooks of the attempt that made them, whatever its outcome, and MUST be committed to the data bag only when the attempt succeeds. The contributions of a failed attempt, retriable or not, MUST NOT reach later steps or the result.
6. **Overwriting.** A committed contribution MAY replace a key already in the data bag, including one from the initial data. The last committed value wins, and `data_overwritten` is emitted.
7. **Handles are valid only during their attempt.** During its attempt, a step contributes and emits freely, in any order. Its contributor and step reporter are made by the executor for that attempt and closed when the attempt ends; closing is idempotent. A call on a closed handle runs no custom code and has no effect on any journey: nothing is recorded, shown to hooks or committed, and no event is delivered. The call SHOULD fail where it is made, so the misuse is visible, but that failure MUST NOT change any journey. Implementations SHOULD make such a call impossible to express (chapter 3, 3.8). The same holds for a hook's contributor and its means of emitting events, until the hook returns.

## 4.7 Received data is read-only

1. Every value a step receives as an input is read-only for it.
2. Changing a received value, where the language allows it at all, MUST NOT change the data bag, the contributions of any step or hook, or what any other step or hook receives.
3. The only way to change the data bag is a contributor, so every change is visible in the events.
4. How this is guaranteed is each language's choice, recorded in its tech spec: for example through ownership and borrowing, or by giving each receiver its own copy or an immutable view.

## 4.8 Time

1. **The executor never interrupts custom code that does nothing illegal.** It does not interrupt a step's execution at any stage (building it, running it, its events, or the hooks after it), and it does not cancel futures, stop threads or impose time limits on a step while it runs.
2. **When custom code does something illegal** and the executor learns of it, the executor aborts the journey. Where an executor-owned signal, given at a call the step or hook itself made, is the only way to stop it, the executor ends it there (chapter 2, 2.9 point 5); whatever the code does afterwards is ignored, and the abort stands.
3. A step handles its own time limits, for example a timeout on a call it makes, and reports a failure, retriable or not, when it runs out of time.

## 4.9 Checked by

In the conformance repository: [`0002-foundations/data-from-the-workflow.feature`](https://github.com/itinera-dev/conformance/blob/main/cases/tier-1/0002-foundations/data-from-the-workflow.feature), [`0008-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0008-steps) (building, outcomes, skipped and contributions), [`0024-abnormal-termination-retriable/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0024-abnormal-termination-retriable) [`0027-received-data-read-only/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0027-received-data-read-only), [`0056-data-bag-values/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0056-data-bag-values), [`0057-handles-valid-during-their-attempt/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0057-handles-valid-during-their-attempt) and [`0060-input-adapters-attached-to-steps/`](https://github.com/itinera-dev/conformance/tree/main/cases/tier-1/0060-input-adapters-attached-to-steps).
