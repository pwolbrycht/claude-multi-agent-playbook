---
name: fan-out
description: Rules for running many Claude Code agents without stalling. Use when planning or debugging a fan-out of subagents, a Workflow script, an agent team, or when a multi-agent session crawls, hits 429/529 back-off, or burns the usage window. Covers concurrency caps, stubs-plus-files handoff, scripted merges, cache settings and diagnosis.
---

Read `RULES.md` in this directory and apply it. The diagnosis section comes first: a stalled fan-out is usually
request back-off, not a full context window. For the evidence behind any rule, read `report.md`; the sources are in
`notes/`.
