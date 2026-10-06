# 0063: The executor is given a dispatcher factory

- Issue: [#63](https://github.com/itinera-dev/spec/issues/63)
- Tier: 1
- Amends: [0011](0011-events.md), [0012](0012-local-executor.md)

## Summary

An executor is given a **dispatcher factory** instead of a dispatcher. It calls the factory once per journey, so every journey has its own dispatcher, and nothing needs to be removed from a dispatcher when a journey ends.

Amends proposal 0011 (items 3, 4 and 6, spec 2.2) and proposal 0012 (item 8, spec 5.9).

## Motivation

Spec 2.2 gives a dispatcher two operations, add a reporter and dispatch an event, and also requires that the workflow's reporters added for a journey be removed from a dispatcher given to the executor when that journey ends (2.2 point 5, 5.9 point 4). Nothing can remove them, and the only workaround, wrapping each reporter and switching it off, leaves a dead wrapper per reporter per journey for as long as the process runs.

Adding a third operation, clear, was considered. A factory is simpler: the default dispatcher is already created afresh for each journey, and giving the executor a way to create every dispatcher the same way leaves nothing to clear.

## Design

1. **The executor is given a dispatcher factory**, or uses the default one. A dispatcher factory creates a dispatcher. The default factory creates the default dispatcher.
2. **One dispatcher per journey.** For each journey, the executor MUST call the factory once, before the journey starts, add the workflow's reporters to the dispatcher it returns, and dispatch every event of the journey through it. When the journey ends, the executor drops it, with every reporter added to it.
3. **The dispatcher keeps its two operations**: add a reporter, and dispatch an event. What adding a reporter means, and how a dispatcher delivers, is still decided by the dispatcher; for example, a test's factory may return a new dispatcher each time, holding the same shared recording reporter and ignoring the workflow's reporters.
4. **A factory that fails** while creating a dispatcher is a refusal, before the journey starts: no journey, no event. It joins the table of custom code in 5.10, next to a dispatcher that fails while reporters are added.
5. **This replaces** the dispatcher "given to the executor when it is created" (2.2 point 2) and the rule that the workflow's reporters must be removed from it (2.2 point 5); 5.9 point 6 ("one dispatcher per journey") now holds for every executor.

## Events

See the design.

## Conformance

The sentences "the executor is given a dispatcher holding the reporter ..." mean that the executor is given a factory whose dispatchers hold that same reporter. The scenario "A dispatcher given to the executor leaks no reporter from one journey to the next" (`0012-local-executor`) keeps checking the outcome, and a new scenario checks that a factory that fails is a refusal, with no event.

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0063-dispatcher-factory/`, with the amended scenarios of earlier proposals tagged `@proposal-0063`.

## Alternatives considered

- **A third operation, clear, called when each journey ends.** Rejected: it needs a rule for which reporters it removes and a rule for when it fails, while a factory needs neither.
- **Wrapping each reporter and switching it off.** Rejected: it leaks a dead wrapper per reporter per journey.

## Open questions

None.
