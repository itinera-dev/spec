# 0083: What a failure, an abort and a refusal carry

- Issue: [#83](https://github.com/itinera-dev/spec/issues/83)
- Tier: 1
- Amends: [0065](0065-business-result.md), [0040](0040-events-facts-and-decisions.md), [0024](0024-abnormal-termination-retriable.md), [0008](0008-steps.md), [0010](0010-hooks-lifecycles-and-roles.md) and [0032](0032-configuration-errors.md)

## Summary

One rule for what a failure, an abort and a refusal carry, everywhere they are reported:

1. **A failure** always carries its **cause**, a **reason** when a step or hook wrote one, and an **error** when an error ended the attempt.
2. **An abort** carries its abort reason and details, and an **error** when custom code caused it by failing.
3. **A refusal** caused by failing custom code carries that error.
4. **Events carry an error only as its message**, as text. The result carries the message, and the error itself where the language can hold any error in one value.
5. **The specification defines what is carried, never its shape**: each language decides whether it uses a tagged union, optional fields or anything else.

Amends proposal 0065 (the result), proposal 0040 (the events `journey_failed` and `journey_aborted`), proposal 0024 (abnormal termination retriable), proposal 0008 (item 10) and proposal 0010 (item 9), on what hooks may request, and proposal 0032, on the details of `journey_aborted`. In the chapters it changes 2.5, 2.7, 5.8, 5.10 and 6.6 point 1.

## Motivation

Proposal 0024 lets an abnormal termination be retried within the retry budget, and when the budget runs out the cause is `retries exhausted`. The result (0065) and `journey_failed` (0040) then carry "the reason of the last failure". When the last attempt ended in an abnormal termination there is no reason, only an error, so the specification leaves this case undefined.

The gap is wider than that case. `journey_failed` carries "its cause and reason" even when the cause is `abnormal termination`, which has no reason. `on step failure` and `on step retry`, called after an abnormal termination, may request "the failure's reason" (6.6 point 1), which does not exist. And an abort or a refusal caused by failing custom code (`hook failed`, `reporter failed`, `step could not be built`, `policy could not be built`, a dispatcher factory that fails) carries nothing that says what went wrong, so whoever inspects the journey cannot see the error that caused it.

A step fails for a reason: because it chose to, because its attempts ran out, or because an abnormal termination left no budget, and then the error is part of that reason. One rule, applied wherever a failure is reported, covers every case.

## Design

### Failures

1. **What a failure carries.** A failed journey, `journey_failed`, and the failure data a hook may request all follow one rule: a failure always carries its **cause**; it carries a **reason** (code, message, details) when a step or hook wrote one; and it carries an **error** when an error ended the attempt.

   | Cause | Reason | Error |
   |---|---|---|
   | `failure` | the step's reason | none |
   | `retries exhausted` | the reason of the last attempt, if it reported a retriable failure | the error of the last attempt, if it ended in an abnormal termination |
   | `abnormal termination` | none | the error |
   | `FailWorkflow` | the reason the hook gave | none |

2. **The result.** A failed result MUST carry the cause, and the reason and the error as point 1 says. The text of 5.8 for a failed journey becomes: "**failed**, with its cause, the reason when a step or hook gave one, and the error when an error ended the last attempt (2.7 point 4)".
3. **`journey_failed`.** It MUST carry the step that failed, its cause, the reason when there is one, and the error's message when there is an error, as point 1 says; or, for `FailWorkflow`, the step whose hook returned it and that reason. It replaces "its cause and reason" in 2.7.
4. **What hooks may request.** The rule of point 1 holds for every hook and every cause. A step hook MAY request the failure's reason and the error. A reason exists when the attempt reported a failure; an error exists when the attempt ended in an abnormal termination. A required request for one that does not exist MUST abort the journey with `required data missing`; an optional request is answered with it absent.

### Aborts and refusals

5. **What an abort carries.** An aborted journey, in `journey_aborted` and in the result, carries its abort reason and its details, and, when custom code caused the abort by failing (by throwing or by returning an error), that **error**. This covers `hook failed`; `reporter failed`, for a reporter or for a dispatcher that fails while dispatching; and `step could not be built` and `policy could not be built` when a constructor, an input adapter or a policy failed. An abort that no error caused, such as `required data missing`, `wrong type`, `not a value`, or `step could not be built` because something supplied is not a value, carries no error.
6. **What a refusal carries.** A refusal caused by failing custom code (a dispatcher factory that fails, a dispatcher that fails while the journey's reporters are added, a workflow policy that cannot be built) carries that error.

### How an error is carried

7. **Events carry the message.** `journey_failed` and `journey_aborted` carry the error only as its message, as text, never the error itself: reporters serialize events, and an error cannot be serialized in general.
8. **The result and refusals carry the error.** Wherever the result or a refusal carries an error, it MUST carry its message, and SHOULD carry the error itself, so that the caller can inspect it, in every language that can hold any error in one value.
9. **The error's message** is the language's own text for the error. When the error comes from a failure scripted with a message, as conformance does, the message is exactly that text, with nothing added.

### Events about the journey, and shape

10. **`journey_failed` and `journey_aborted` are events about the journey**, not events about a step (2.5 point 2). They explain why the journey ended: `journey_failed` names the step that failed, and `journey_aborted` the step during which it was aborted, if any, but neither carries an attempt number.
11. **Content, not shape.** The specification lists what a succeeded, failed or aborted result, and a refusal, carry. Whether a language represents them with a tagged union, optional fields or anything else is its own choice.
12. **Unchanged.** `step_given_up` keeps carrying only the cause; `step_abnormal_termination` keeps carrying the error's message; a failure while `journey_aborted` is delivered is still ignored and adds nothing to it.

## Events

No new events. `journey_failed` and `journey_aborted` carry the error's message where the design says, and name their step without an attempt number.

## Conformance

The conformance cases for this proposal must check that:

1. A step whose abnormal terminations are retriable, with a retry budget of 1, ends both attempts in an abnormal termination with the message "timeout": `journey_failed` carries the cause `retries exhausted` and the error "timeout"; the result's failure has that cause, that error and no reason.
2. The same step's failure hook requests the error and receives "timeout".
3. The same step's failure hook requests the failure's reason as required: the journey is aborted with `required data missing`.
4. A retry budget of 1, where the first attempt ends in an abnormal termination and the second reports a retriable failure with the code "busy": the result's failure has the cause `retries exhausted`, the reason "busy" and no error.
5. An abnormal termination that is not retriable, with the message "boom": `journey_failed` carries the cause `abnormal termination` and the error "boom" and names the step; the result has that cause, that error and no reason.
6. A retry hook called after a retriable abnormal termination requests the failure's reason as optional, and receives it absent.
7. A hook that fails with the message "ledger closed": `journey_aborted` and the result carry `hook failed` with the error "ledger closed".
8. A reporter that fails with the message "disk full": `journey_aborted` carries `reporter failed` with the error "disk full".
9. An abort for `required data missing` carries no error.
10. A workflow policy that fails with the message "no configuration" when it is built: `run` is refused with that message.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), tagged `@proposal-0083`, with the cases of the proposals they amend. FORMAT.md defines the `error` column of event tables.

## Alternatives considered

- **Wrap the error message in a reason** with a fixed code such as `abnormal termination`. Every failure would have the same shape, but the specification would invent a reason code no step chose, and a reader could not tell a real reason from a wrapped one. The cause already says why.
- **Report the cause `abnormal termination`** when the last attempt ended abnormally after retries. That contradicts 0024, which names `retries exhausted`, and hides that retries happened.
- **Fix only `retries exhausted`.** It would leave `journey_failed` and the hooks without a rule for the cause `abnormal termination`.
- **Carry the error itself in events too.** Events are serialized by reporters, and an error cannot be serialized in general, so events carry its message.
- **Carry only the message in the result.** It is enough to show what went wrong, but a caller could not inspect the error, for example to tell one kind of failure from another.
- **Require the error in every abort.** Some aborts have no error behind them, so it would be invented.

## Open questions

None.
