# Behaviour specification

The normative behaviour specification of Itinera: exactly what every implementation must do, in terms the conformance suite can observe. Its chapters are written from accepted proposals in [../proposals/](../proposals/), and use the key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY as described in RFC 2119.

Nothing here is specific to one language. How each language implements these chapters is described in that language's tech specs, in its own repository. See "The three kinds of document" in [PROCESS.md](../PROCESS.md).

No chapters exist yet. The first ones will be written from the core model proposal ([#2](https://github.com/itinera-dev/spec/issues/2)) and will make up version 0.1.

Planned chapters, which may change as proposals are accepted:

1. Concepts: what each concept is and how they relate. The chapter to read first, and the single source of the vocabulary.
2. Steps
3. Workflows and the executor's scan
4. Step statuses and retries
5. Hooks and lifecycles
6. The data bag and adapters
7. Events
8. Executors, capabilities and admission
9. The workflow result
