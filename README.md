# Itinera specification

Itinera is a workflow framework that keeps business rules separate from flow control. Steps are business units that never know which workflow runs them. A workflow declares, in one place, the order of its steps and the policies that decide what happens after each outcome. An executor runs it and reports every event along the way.

This repository holds the language-neutral specification. Every Itinera implementation, starting with [itinera-rs](https://github.com/itinera-dev/itinera-rs), follows it, and the [conformance suite](https://github.com/itinera-dev/conformance) checks that they do.

## What is here

- [VISION.md](VISION.md): why Itinera exists and the principles it keeps.
- [PROCESS.md](PROCESS.md): how features are proposed, evaluated, specified and tracked across languages.
- [ROADMAP.md](ROADMAP.md): the planned tiers and milestones.
- [proposals/](proposals/): accepted proposals, one file per proposal, numbered by its issue.
- [spec/](spec/): the normative specification, written from accepted proposals.
- [AGENTS.md](AGENTS.md): the working manual for anyone, human or agent, working in this repository.

## Proposing a feature

Open a Proposal issue in this repository. [PROCESS.md](PROCESS.md) describes what happens next.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
