# claude-multi-agent-playbook

Field-tested rules for running Claude Code at scale, kept in git so every machine and every project session loads
the same ones.

## What's inside

- `fan-out/RULES.md`: how to run many agents without stalling. Concurrency caps, stubs plus files, scripted fan-out.
- `fan-out/SKILL.md`: makes the rules a Claude Code skill, loaded only when a task involves agents or workflows.
- `fan-out/report.md` and `fan-out/notes/`: the research behind the rules (Anthropic docs, GitHub issues,
  practitioner reports and the literature, October 2026).
- `settings/user-settings-snippet.json`: recommended user settings. A 1-hour subagent cache TTL and concurrency caps.
- `INSTALL.md`: the procedure. No scripts to run or audit.

## Install

Clone the repository, then follow [INSTALL.md](INSTALL.md) yourself or hand it to your agent:

> Read INSTALL.md in this repository and apply it on this machine.

Everything it does is additive and listed there, with an uninstall section.

## Why

On a Claude Max plan a multi-agent fan-out stalls because of request back-off and cache churn, not because the
orchestrator's context is full. The rules file is the short version. The report has the evidence, the trade-offs and
the places where the evidence is thin.

## License

[MIT](LICENSE)
