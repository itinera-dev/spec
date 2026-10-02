# 0012: The local executor

- Issue: [#12](https://github.com/itinera-dev/spec/issues/12)
- Tier: 1
- Amended by [#42](https://github.com/itinera-dev/spec/issues/42): configuration errors are caught before a journey starts.

## Summary

What an executor is, how it is called, the guarantee that nothing leaks between journeys, and the tier 1 capabilities. Builds on proposals 0002, 0008, 0009, 0010, 0011 and 0032.

## Motivation

Workflow authors need one simple way to run a journey and get its result, and the certainty that running many journeys never mixes their data, statuses or events. Languages differ in what they can run (synchronous or asynchronous code) and in when they can catch configuration errors, so those differences must be declared as capabilities and never change what a workflow means.

## Design

### Executors

1. **Executors differ in capability, never in meaning.** A lifecycle, an outcome, a status or an event MUST mean the same under every executor.
2. **The local executor** runs a journey in process, with no persistence: a journey lives only as long as the call that runs it.

### Running a journey

3. **`run(workflow instance)`** MUST run exactly one journey and return its result: succeeded, failed or aborted. It MUST NOT throw for anything that happens inside the journey; failures and aborts are in the result. An asynchronous executor returns the same result asynchronously.
4. **The executor asks the workflow instance** for its step descriptors, data and reporters.
5. **A workflow instance runs at most one journey.** Handing an instance that has already run, or is running, to an executor MUST be refused as a configuration error.

### Stateless, one journey at a time

6. **An executor is stateless between journeys.** When `run` returns, nothing of that journey MUST remain in the executor. An executor MAY run many journeys, one after another.
7. **One journey at a time.** An executor MUST NOT run two journeys concurrently. A call to `run` while another journey is running on the same executor MUST be refused before anything starts, with no journey and no events. Implementations SHOULD make this impossible to write where their language allows it. For parallelism, several executors are created.
8. **One dispatcher per journey.** The default dispatcher MUST be created afresh for each journey. A dispatcher given to the executor is used for each journey in turn, and the workflow's reporters added for a journey MUST be removed when that journey ends, so they never receive another journey's events.

### Capabilities in tier 1

9. **Execution modes:** `sync` and `async`. An executor declares which modes it accepts, for steps, hooks and reporters alike. Anything it does not accept is a configuration error, with the violation `mode not accepted`: caught when the workflow is built if the language can, otherwise when the journey runs. Whether an asynchronous executor also accepts synchronous steps is each language's choice, stated as a capability.
10. **Configuration checks:** `build-time-configuration-check` or `run-time-configuration-check` (proposal 0032). A language claims exactly one.
11. **A language claims its capabilities** alongside its tier, and its conformance runner runs the cases tagged with them.

## Events

None.

## Conformance

The conformance cases for this proposal must check that:

1. `run` returns the result of a failed or aborted journey instead of throwing;
2. an executor is stateless between two journeys, which have different journey IDs and identical event streams apart from IDs and timestamps;
3. a dispatcher given to the executor leaks no reporter from one journey to the next;
4. a synchronous-only executor refuses an asynchronous step, when the workflow is built or by aborting with `invalid configuration`, according to the language's configuration-check capability.

Running an instance twice and calling `run` concurrently are checked by each language's own tests, since many languages make both impossible to write. The cases are in [itinera-dev/conformance](https://github.com/itinera-dev/conformance), under `cases/tier-1/0012-local-executor/`.

## Alternatives considered

- **Running several journeys concurrently on one executor.** Rejected: an executor runs one journey at a time and is stateless between them; parallelism comes from several executors.
- **A dispatcher shared across journeys with the workflow's reporters kept.** Rejected: a reporter must only ever receive its own journey's events.
- **Executors requiring particular hooks in tier 1.** Deferred to durable runs.

## Open questions

- Durable executors: persistence, resuming after a crash, pending steps, and the hooks they may require.
