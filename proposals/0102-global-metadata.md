# 0102: Global metadata passed to every reporter

- Issue: [#102](https://github.com/itinera-dev/spec/issues/102)
- Tier: 1
- Amends: [0008](0008-steps.md), [0010](0010-hooks-lifecycles-and-roles.md), [0011](0011-events.md), [0054](0054-how-custom-code-fails.md), [0061](0061-instance-carries-journey-id-and-reporters.md), [0063](0063-dispatcher-factory.md) and [0095](0095-services-provided-to-steps.md)

## Summary

Every journey has **global metadata**: entries of a key and a value, kept apart from the data bag, that the dispatcher passes to every reporter as soon as each one is set. The journey ID is always its first entry. The workflow instance may start with more entries, and steps, hooks and service providers may set entries while the journey runs. Each reporter keeps the entries and decides what to do with them, for example writing them with everything it reports about the journey's later events.

Amends proposal 0011, in what reporters and dispatchers do and in the engine catalogue, which gains the fact `global_metadata_set`. Amends proposal 0063, in the operations of a dispatcher. Amends proposal 0061, in how reporters receive the journey ID. Amends proposals 0008, 0010 and 0095, in what a step reporter, a hook and a service provider may do. Amends proposal 0054, in the list of calls into custom code and their outcomes. In the chapters it changes chapter 1, 2.1, 2.2, 2.3, 2.7, 2.8, 2.9, 3.6, 3.7, 4.1, 4.6 point 7, 4.9 point 7, 5.1, 5.7, 5.10 and chapter 6, where hooks emit events.

## Motivation

Reporters write a journey somewhere: a log, a trace, an audit trail. Whoever reads what they wrote needs to know whom and what the journey was for: the user, the tenant, the order. A journey often learns these only while it runs, for example when a step looks up the user.

Today a reporter receives only events. Engine events never carry data values (2.6 point 4), and a step can put a value only into its own `step_*` events. To write a user ID with every later event, each reporter would have to know which step emits it, under which message, and remember it. Reporters are adapters, often generic ones written once for every workflow, so that convention does not scale.

Global metadata gives reporters one operation for it. The workflow decides what the reporters receive; each reporter decides what to do with it.

## Design

### Global metadata

1. **What it is.** The global metadata of a journey is a set of entries, each a **key**, a string, and a **value** (4.6). It belongs to the journey and exists for its reporters.
2. **Apart from the data bag.** It never enters the data bag or the result. No step, hook, input adapter or service provider can read it.
3. **One entry per key.** Setting a key that already has an entry replaces its value. An entry is never removed.
4. **The journey ID.** The key `journey_id` holds the journey ID, as a string. Only the executor sets it.

### Reporters and dispatchers

1. **A reporter has a second operation, receive global metadata,** which receives one entry. It MUST accept it, and decides on its own what to do with it. A reporter SHOULD keep the latest value of each key until the journey ends, so that it can use them with the events that follow.
2. **A dispatcher has a third operation, set global metadata,** which receives one entry and passes it on to the reporters it holds.
3. **The default dispatcher** MUST pass every entry to every reporter it holds, in the order they were added, except as the failure rules below say. Other dispatchers decide what setting global metadata means, as they do for adding a reporter and for dispatching (2.2 point 5).
4. **In order with the events.** Entries and events reach reporters in the order they happen. An entry set while the journey runs reaches the reporters after every event emitted before it was set and before every event emitted after it.

### Before the journey starts

1. **Initial global metadata.** The workflow instance MAY be created with initial global metadata, as it may with initial data (5.1). Keys are strings and every value MUST be a value. A key set twice while the instance is created keeps its last value.
2. **Set before the first event.** After adding the reporters to the dispatcher (2.2 point 3) and before `journey_started`, the executor MUST set `journey_id` to the journey ID, then each entry of the initial global metadata, in the order its keys were first set. Neither emits an event.
3. **Refused before the journey.** These are refusals by `run`, with no journey and no event (3.7, 5.10):
   1. initial global metadata that is not a value, or that sets `journey_id`, listing each key concerned;
   2. a reporter, or the dispatcher, that fails while the initial global metadata, the journey ID included, is set.

### While the journey runs

1. **Who may set it.** A step, through its step reporter, and a hook and a service provider, through their means of emitting events, MAY set global metadata, one entry at a time. A rescue step is a step. Input adapters, role operations and constructors cannot.
2. **Captured when set.** The value is captured when it is set: changing the original afterwards changes nothing.
3. **Delivered before the call returns.** The entry is passed to the dispatcher, and so to the reporters, and then `global_metadata_set` is emitted, all before the call returns. The call blocks for synchronous custom code and is awaited for asynchronous custom code.
4. **At once, and for good.** An entry stands from the moment it is set, whatever the code that set it does afterwards. A failed, retried or skipped attempt, an abnormal termination and an abort take back nothing, as events are never rolled back (2.4).
5. **A value that is not a value** aborts the journey with `not a value`, naming the step, the policy and hook, or the service provider and the step being built, and the key. The entry reaches no reporter, `global_metadata_set` is not emitted, and the call ends the step, hook or provider as for a reporter that fails (2.9 point 5).
6. **The key `journey_id`** aborts the journey with the new abort reason `reserved metadata key`, naming the same and handled the same way as point 5. It is a configuration error that can only show up during a journey.
7. **A reporter or the dispatcher that fails** on the entry aborts the journey with `reporter failed`, exactly as on an event (2.3): the entry reaches no reporter after the one that failed, `global_metadata_set` is not emitted, `journey_aborted` reaches every reporter except the one that failed, and the call ends the step, hook or provider (2.9 point 5).
8. **Closed handles.** Setting global metadata through a step reporter whose attempt has ended, or through a hook's or provider's means of emitting events after its call has returned, follows 4.6 point 7: it has no effect on any journey.
9. **When the journey ends,** the global metadata goes with the dispatcher and its reporters (2.2 point 3).

### Unchanged

1. Every event still carries the journey ID among its fixed fields (2.5).
2. Engine events still carry keys, never values (2.6 point 4): `global_metadata_set` carries the key only.
3. A reporter MAY still be made with the journey ID while the instance is created (5.1).

## Events

One new fact in the engine catalogue (2.7):

| Event | When | Also carries |
|---|---|---|
| `global_metadata_set` | a step, hook or service provider set an entry of global metadata, after the reporters received it | the key, and who set it: the step and attempt; the policy and hook, with the step and attempt for a step hook; or the service provider, with the step and attempt of the step being built |

In the order of events (2.8), `global_metadata_set` comes where it is set: among a step's own `step_*` events, among the `journey_*` events a hook emits, and among the `journey_*` events a service provider emits, in the order they happen. Setting `journey_id` and the initial global metadata emits nothing.

## Conformance

The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0102-global-metadata/`. They check that:

1. Every reporter receives the journey ID as its first entry, before any event.
2. The initial global metadata reaches the reporters after the journey ID, in the order it was set, before any event.
3. An entry a step sets reaches the reporters when it is set, and `global_metadata_set` records the key and the step, but not the value.
4. An entry set by an attempt that then fails stays: nothing withdraws it.
5. Setting a key again passes the new value to the reporters.
6. An entry a hook sets records the policy and hook as its source.
7. An entry a service provider sets records the provider.
8. The default dispatcher passes every entry to every reporter.
9. A step's entry that is not a value aborts the journey with `not a value`, and reaches no reporter.
10. Initial global metadata that is not a value is refused by `run`, with no event.
11. A step that sets `journey_id` aborts the journey with `reserved metadata key`.
12. Initial global metadata that sets `journey_id` is refused by `run`, with no event.
13. A reporter that fails on an entry aborts the journey with `reporter failed`: the reporters after it do not receive the entry, and `journey_aborted` reaches them.
14. A reporter that fails on the journey ID makes `run` refuse the journey, with no event.

## Alternatives considered

- **Declared keys of the data bag, carried on every event.** Rejected by the maintainer: data stays out of events, and each reporter decides what to do with what it receives.
- **A copy of the global metadata on every event, written by the executor.** Rejected: reporters receive each entry once and keep it as they choose.
- **Entries delivered only when the attempt that set them succeeds.** Rejected: a user ID a step retrieved stays true even if the attempt fails afterwards, and delivering at once keeps the rule simple.
- **A convention that reporters remember data from `step_info`.** Rejected: every reporter would need to know every workflow's conventions.

## Open questions

None.
