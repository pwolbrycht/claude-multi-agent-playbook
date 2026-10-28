# Fan-out rules: how to run many agents without stalling

Researched 2026-10-28 from Anthropic docs, GitHub issues, practitioner reports and the literature
(full report: `fan-out/report.md`, sources: `fan-out/notes/`). Applies to every Claude Code session on a Max plan.

## Diagnosis first

- A fan-out that "stalls" at a low token rate is **requests not completing** (429/529 retry back-off), not a full
  context window. A bloated orchestrator raises tokens per minute; it does not collapse them.
- Max users measured throttling from **~6 concurrent streams** while plan usage read 20-30 %.
- Compounding: subagents get a **5-minute** cache TTL by default (parent gets 1 h); every background completion
  notification re-sends the whole parent context; peak hours are 5-11 am Pacific.
- When a session crawls: `/usage` first. Cold cache = churn (tool list, model or compaction changed). Warm cache with
  no progress = back-off. A silent worker: read its transcript under `~/.claude/projects/<project>/<session>/subagents/`
  instead of waiting.

## Rules

1. **Concurrency, not totals, is the limit.** Hold concurrent streams to **3-5 per wave**, starts staggered a few
   seconds. Total agent count can be whatever the job needs.
2. **The orchestrator never reads raw output.** Workers write full output to a run directory and return a stub of
   **≤ 1,500 tokens plus the path**. Every brief carries: "if the Write fails, say so and return the path you tried,
   never the content."
3. **More than ~10 workers: a Workflow script owns the loop.** Results stay in script variables; `schema` on each
   `agent()` for the stub; prose reasoning inside the worker. Accept no mid-run steering. Run a small slice first.
4. **Merge in code where possible**, otherwise one fresh agent reads the files. Never the orchestrator.
5. **Parallel reads, serial writes.** Fan out readers and auditors; one agent does the edits.
6. **Fresh briefed agents, not forks**, once the parent is large. Readers on Haiku/Sonnet with `maxTurns`; the
   built-in Explore agent for lookups.
7. **Never switch model, effort or fast mode mid-run** in the orchestrating session (cache invalidation re-bills the
   whole prefix).
8. **Between phases: handoff file + `/clear`**, not `/compact` (compact is itself a large request).
9. **Keep raw findings on disk** so a limit hit or a dead run loses nothing and resumes from files.
10. **Measure**: tokens per wave, 429s per minute (Background tasks pane), cache hit share (`/usage`). Record them.

## Settings (see INSTALL.md; snippet in settings/user-settings-snippet.json)

- `~/.claude/settings.json`: `"subagentPromptCacheTtl": "1h"`
- Environment: `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS=5`, `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS=4`

## Known gaps

No controlled comparison of these patterns exists; the Max concurrency ceiling is inferred from user-filed issues
Anthropic has not answered; the hook that would enforce stub size on Agent results is untested.
