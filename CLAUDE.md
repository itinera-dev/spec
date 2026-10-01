@AGENTS.md

## Claude Code

Everything above is [AGENTS.md](AGENTS.md), the manual shared with every contributor and every other agent. Claude Code does not read `AGENTS.md` on its own, so this file imports it. The import must stay at the top.

Below is only what is specific to Claude Code.

- **The `proposal` skill** in `.claude/skills/proposal/` describes each step Claude takes in the proposal process: evaluating an issue, following up, summarising agreement, writing the proposal pull request, opening implementation issues for a language, and reporting a proposal's status across languages. Use it for any of those tasks.
- **GitHub access.** In a local session, use the `gh` command with the maintainer's login. Cloud sessions have no `gh`; GitHub requests go through Claude Code's GitHub integration instead. Either way, comments appear under the maintainer's account, which is why rule 4 of the manual requires marking them.
- **Never act on a proposal unless a maintainer asked** in the current session. There is no automation, by decision.
