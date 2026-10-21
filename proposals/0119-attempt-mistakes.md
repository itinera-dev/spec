# 0119: An attempt scripted twice, an attempt a step can never have, and reading the record of an unknown step

- Issue: [#119](https://github.com/itinera-dev/spec/issues/119)
- Tier: 1, as the capability `testing`
- Amends: [0108](0108-test-support.md)

## Summary

Chapter 8, 8.5 named the mistakes a test can make while writing a script, but not three that implementations already stop a test for. Choosing an attempt the script has already described is the new mistake `attempt scripted twice`. Choosing an attempt the step can never have, such as attempt 0, a fraction, or one past what its retry budget and rescue step allow, is the new mistake `no such attempt`. Reading a scripted run's record of a step the workflow does not have is the existing mistake `unknown step`, stopping the test where the record is read. When one choice of attempts is both `no such attempt` and `attempt scripted twice` or `gap in the attempts`, only `no such attempt` is named. Reading an attempt a step's record does not hold reads nothing and is not a mistake. No event, abort reason, status or violation is added.

Amends proposal 0108. In the chapters it changes 8.1 point 5 and 8.5 points 8 and 9.

## Motivation

1. **Attempts chosen twice, or never possible.** 8.5 point 8 named a gap in the attempts, but not `on attempt 1` written twice for one step, `on every attempt` after `on attempt 1`, or any choice after `on remaining attempts`. Nor did it name an attempt the step can never have: attempt 0, -1 or 1.5, attempt 3 of a step with a retry budget of 1, attempt 2 of a rescue step, or `on remaining attempts` once every attempt the step can have is described. Point 7 checks only the attempts a script leaves out. An implementation could ignore the second description, let it replace the first, accept an attempt that never runs, or stop the test, and a conformance case could not name the mistake. itinera-rs and itinera-ts both stopped the test, but they worded and ordered these checks differently, so a call that makes two mistakes was reported differently by each.
2. **Reading the record.** 8.5 point 9 said a step's record is read with `step` by name, and an attempt with `attempt` by number, without saying what a name the workflow does not have, or a number the record does not hold, does. 8.1 point 5 said a mistake in a test stops it before the journey starts, which a read made after the journey cannot do.

## Design

1. **`attempt scripted twice`.** Choosing attempts the script has already described for a step MUST stop the test where it is written, as a mistake in the test (8.1 point 5):
   1. `on attempt` with the number of an attempt already described, or with any number once `on every attempt` or `on remaining attempts` was chosen for the step;
   2. `on every attempt` once any attempt of the step is described;
   3. `on remaining attempts` once it, or `on every attempt`, was chosen for the step.

   It names the step and the first attempt the choice describes again: the number given to `on attempt`, attempt 1 for `on every attempt`, and, for `on remaining attempts`, the attempt after the last one described by number, or attempt 1 when none was.
2. **`no such attempt`.** Choosing an attempt the step can never have MUST stop the test where it is written, as a mistake in the test:
   1. `on attempt` with a number that is not a whole number from 1, such as 0, -1 or 1.5;
   2. `on attempt` with a number greater than the most attempts the step can have (chapter 5, 5.6 point 1): its retry budget plus one, and one more for a step with a rescue step, so a rescue step, which has no retry budget, can have one;
   3. `on remaining attempts` once the attempts described by number reach that most.

   It names the step and the attempt: the number given to `on attempt`, and, for `on remaining attempts`, the attempt after the last one described by number. A number that a language's type for an attempt's number cannot hold, such as a fraction where that type holds whole numbers, is impossible to express (proposal 0054), and that language need not check it: it proves this, and lists the scenarios it excludes, as for the rules of chapter 3, 3.8, points 2 and 3.
3. **Which mistake is named.** When one choice of attempts is both `no such attempt` and `attempt scripted twice` or `gap in the attempts`, the test MUST be stopped naming only `no such attempt`. With nothing yet described for a step with a retry budget of 1, `on attempt 5` is `no such attempt`, not a gap; after `on every attempt`, it is `no such attempt` too, not `attempt scripted twice`. An attempt the step can never have stays wrong whatever else the script describes. A choice is never both `attempt scripted twice` and `gap in the attempts`, since the attempts before one already described are described too.
4. **Reading the record of a step the workflow does not have.** Reading a scripted run's record with `step` and a name that is neither a step nor a rescue step of the workflow MUST stop the test where it is read, as the mistake `unknown step`, naming that name. It happens after the journey, which stays as it ended.
5. **Reading an attempt the record does not hold.** Reading a step's record with `attempt` and the number of an attempt it does not hold, such as one its step never reached, or 0, MUST read nothing, as the language says a value is absent. It is not a mistake: whether an attempt happened is what a test reads the record to learn. A number that a language's type for an attempt's number cannot hold is impossible to express here too, as in point 2.
6. **8.1 point 5** says that a mistake in a test MUST stop the test where it is made: before the journey starts for a mistake in a fake provider or a script, and where the record is read for a mistake in reading it. It is still not a refusal, no event is emitted, and a journey that already ran is unchanged.
7. **8.5 point 8** names an attempt chosen twice and an attempt the step can never have among the mistakes in writing a script, with point 3; **8.5 point 9** says what `step` and `attempt` do with a name or a number the record does not hold.

Nothing else changes: how a scripted journey runs, the check before the journey (8.5 point 7), the events and the result are as before.

## Events

None added or changed.

## Conformance

The cases live with those of proposal 0108, which this one amends, tagged `@proposal-0119`, in itinera-dev/conformance: `cases/tier-1/0108-test-support/script-checks.feature` and `cases/tier-1/0108-test-support/scripted-run-record.feature`, which carry `@capability-testing`. They check that:

1. `on attempt 1` written twice for a step stops the test, naming `attempt scripted twice` with the step and attempt 1, and no event is emitted;
2. `on every attempt` after `on attempt 1` names `attempt scripted twice` with attempt 1;
3. `on attempt 2` after `on every attempt` names `attempt scripted twice` with attempt 2;
4. `on remaining attempts` written twice, after `on attempt 1`, names `attempt scripted twice` with attempt 2;
5. `on attempt 0` names `no such attempt` with attempt 0, in a scenario tagged `@attempt-zero`;
6. `on attempt -1` names `no such attempt` with attempt -1, in a scenario tagged `@attempt-negative`, and `on attempt 1.5` with attempt 1.5, in one tagged `@attempt-fraction`;
7. `on attempt 3` for a step with a retry budget of 1 names `no such attempt` with attempt 3;
8. `on attempt 3` for a step with a rescue step names `no such attempt` with attempt 3;
9. `on attempt 2` for a rescue step names `no such attempt` with the rescue step and attempt 2;
10. `on remaining attempts` after attempts 1 and 2 of a step with a retry budget of 1 names `no such attempt` with attempt 3;
11. `on attempt 5` as the first choice for a step with a retry budget of 1 names only `no such attempt`, not `gap in the attempts`;
12. `on attempt 5` after `on every attempt` for a step with a retry budget of 1 names only `no such attempt`, not `attempt scripted twice`;
13. `on attempt 3` after `on attempt 1` and `on remaining attempts` for a step with a retry budget of 2 names `attempt scripted twice` with attempt 3;
14. reading the record of a step the workflow does not have, after the journey succeeded, stops the test naming `unknown step` with that name;
15. reading attempt 2 of a step whose record holds only attempt 1 reads nothing, and so does reading attempt 0, in a scenario tagged `@attempt-zero`.

STEPS.md gains the mistakes `attempt scripted twice` and `no such attempt`, widens `unknown step` to reading the record, lets `attempt` in "the script says" hold any number, and gains a sentence for reading the record of a step that stops the test and one for an attempt the record does not hold. FORMAT.md gains the tags `@attempt-zero`, `@attempt-negative` and `@attempt-fraction` among the rules a language may make impossible to express. No existing case changes.

## Alternatives considered

1. **A later description replaces the earlier one,** or the second is ignored. Rejected: either way the script no longer says what it reads as saying, and a test passes or fails for a reason it does not show.
2. **Leave an attempt the step can never have unchecked.** Such a description never runs, so it is harmless to the journey. Rejected: it is almost always a miscounted retry budget, and the test then checks attempts other than those its author meant.
3. **Name every mistake one choice makes,** such as both `no such attempt` and `gap in the attempts` for `on attempt 5`. Rejected: the mistakes are reported where they are written, one call at a time, and an attempt the step can never have is the one that must be fixed first.
4. **A new mistake for reading the record of an unknown step,** such as `unknown record`. Rejected: it is the same slip as naming an unknown step in the script, and `unknown step` already names it.
5. **Make reading an attempt the record does not hold a mistake.** Rejected: a test reads the record to learn which attempts happened, and absence is an answer.

## Open questions

None.
