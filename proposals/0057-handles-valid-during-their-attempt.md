# 0057: Contributors and step reporters are valid only during their attempt or hook

- Issue: [#57](https://github.com/itinera-dev/spec/issues/57)
- Tier: 1
- Amends: [0008](0008-steps.md), [0010](0010-hooks-lifecycles-and-roles.md)

## Summary

A contributor or a reporter given to a step is valid only during the attempt it was given for. One given to a hook is valid only until the hook returns. Using it afterwards has no effect on the journey: nothing is contributed and no event is emitted. The call SHOULD fail where it is made, so the misuse is visible, and implementations SHOULD make it impossible to express where their language allows.

Amends proposal 0008 (contributions and the step reporter) and proposal 0010 (a hook's contributor and events).

## Motivation

A step's constructor receives what it asks for, including a contributor and a step reporter, and the step keeps them for its own use. Nothing stops it from handing them to code that outlives the attempt, such as a timer callback, a promise that is never awaited, or a background thread. That code could then contribute or emit after the attempt ended: in the middle of the hooks' events, after the step decision, in a later attempt's events, or after the journey has ended. The same holds for a hook that keeps its contributor or the way it emits events.

The specification does not say what happens then, and it is observable. A late contribution could reach the data bag without being part of any committed attempt, and a late event would break the order of 2.8, or follow `journey_succeeded`, `journey_failed` or `journey_aborted`.

## Design

1. **Validity.** A contributor and a step reporter given to a step are valid from the moment its constructor receives them until its attempt ends: the step returns, fails or terminates abnormally. A contributor given to a hook, and the hook's means of emitting events, are valid until the hook returns.
2. **Use after the end has no effect.** A contribution made through a handle that is no longer valid MUST NOT be recorded, shown to hooks or committed. An event emitted through one MUST NOT be delivered. No engine event records the attempt either.
3. **It is visible to the author.** The call SHOULD fail where it is made, for example by throwing in the code that made it. That failure is outside the journey: it MUST NOT abort, fail or otherwise change any journey, which may already have moved on or ended.
4. **Handles are closed by the executor.** The contributor and step reporter a step receives, and a hook's contributor and means of emitting events, are made by the executor for that attempt or hook, not by the developer. The executor closes them when the attempt ends or the hook returns; closing is idempotent. Every call goes through a check: while the handle is open, the contribution is recorded or the event dispatched, which calls the reporters; once it is closed, no custom code runs and nothing reaches any journey. The reporter interface developers implement keeps its single `report` operation.
5. **Impossible where the language allows.** Implementations SHOULD make using a handle after its attempt or hook impossible to express, for example through lifetimes. This joins the rules a language may make impossible to express (#54), under the tag `@late-handle`.

## Events

See the design.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0057-handles-valid-during-their-attempt/`, with the amended scenarios of earlier proposals tagged `@proposal-0057`.

## Alternatives considered

- **Ignoring late calls silently.** Rejected as the only rule: it hides a bug in the step. The SHOULD in point 3 makes it visible.
- **Aborting the journey on a late call.** Rejected: the journey may have ended, or moved on to unrelated steps, and the call is not on its path.
- **Leaving it unspecified.** Rejected: late contributions and events are observable and would break the order and content of the event stream.

## Open questions

None.
