# Itinera specification

Itinera is a workflow framework that keeps business rules separate from flow control. Steps are business units that never know which workflow runs them. A workflow declares, in one place, the order of its steps and the policies that decide what happens after each outcome. An executor runs it and reports every event along the way.

This repository holds the language-neutral specification. Every Itinera implementation, starting with [itinera-rs](https://github.com/itinera-dev/itinera-rs), follows it, and the [conformance suite](https://github.com/itinera-dev/conformance) checks that they do.

## The rules of the game

Every feature is described by three kinds of document, and only two of them live here.

1. **The proposal, a PRD.** Why the feature exists and what is wanted. Discussed in an issue in this repository, then kept under [proposals/](proposals/) once accepted. Never specific to a language.
2. **The behaviour specification.** Exactly what every implementation must do, in terms anyone can observe: outcomes, step statuses, order, hooks and what they may return, events. Kept under [spec/](spec/) and checked by the conformance suite. Never specific to a language.
3. **The tech spec.** How one language implements a proposal: its API, types, modules and tests. Agreed in that language's implementation issue and kept next to the code, in that language's repository, under `docs/specs/`. **Never in this repository.**

The test for where something belongs: if the conformance suite could observe it, it belongs in the behaviour specification; if it is about how code is written in one language, it belongs in that language's tech spec.

Two more rules:

- **The specification comes first.** No implementation pull request is opened before its proposal is accepted. Implementation ideas are welcome in the proposal issue.
- **When a proposal and the behaviour specification disagree, the specification is right.** Proposals are history; the specification is the current picture.

[PROCESS.md](PROCESS.md) has the details.

**If this looks like bureaucracy:** all that is asked of you is one Proposal issue saying what you need and why. Everything after it, from evaluation to code in every language, is done by the maintainers with agents when the proposal is planned, and you can join wherever you like. Why the documents exist at all is explained in [This is not bureaucracy](https://github.com/itinera-dev/.github/blob/main/CONTRIBUTING.md#this-is-not-bureaucracy).

## Start here

Read [spec/01-concepts.md](spec/01-concepts.md) first: what steps, workflows, hooks, lifecycles, executors, journeys and runs are, and how they fit together. The other chapters of the specification are listed in [spec/README.md](spec/README.md).

## What is here

- [VISION.md](VISION.md): why Itinera exists and the principles it keeps.
- [PROCESS.md](PROCESS.md): how features are proposed, evaluated, specified and tracked across languages.
- [ROADMAP.md](ROADMAP.md): the planned tiers and milestones.
- [proposals/](proposals/): accepted proposals (the PRDs), one file per proposal, numbered by its issue.
- [spec/](spec/): the behaviour specification, normative, written from accepted proposals.
- [AGENTS.md](AGENTS.md): the working manual for anyone, human or agent, working in this repository.

## Proposing a feature

Open a Proposal issue in this repository. [PROCESS.md](PROCESS.md) describes what happens next.

Built with AI under the terms of [A manifesto for software engineering with AI](https://marlon-sousa.com/blog/manifesto/); see [how Itinera is built](https://github.com/itinera-dev/.github/blob/main/CONTRIBUTING.md#how-itinera-is-built).

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
