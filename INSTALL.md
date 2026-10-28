# Install

This repository contains no scripts. Installation is a short procedure that you, or a coding agent you ask, carry out
by hand. A typical prompt is: "Read INSTALL.md in this repository and apply it on this machine."

Nothing here needs root. Every step is additive: existing files are appended to or merged into, never replaced.

## 1. Pick how the rules load

Choose one. The skill is recommended.

**A. As a skill (loads only when relevant, costs nothing otherwise)**

Copy the `fan-out/` directory to the user skills directory:

- Linux and macOS: `~/.claude/skills/fan-out/`
- Windows: `%USERPROFILE%\.claude\skills\fan-out\`

The directory must contain `SKILL.md` and `RULES.md`; the report and notes may come along as references. Claude Code
lists the skill at the start of every session and loads `RULES.md` when a task involves subagents, workflows or a
fan-out that stalls.

**B. As a global instruction (always loaded, in every session)**

Add this line to `~/.claude/CLAUDE.md`, creating the file if it does not exist, using the absolute path of your clone:

    @/absolute/path/to/claude-multi-agent-playbook/fan-out/RULES.md

If the file already exists, append the line. Do not remove anything already in it.

## 2. Merge the recommended settings

Open `~/.claude/settings.json` (create it with `{}` if missing) and merge in the contents of
`settings/user-settings-snippet.json`:

- Add `subagentPromptCacheTtl` at the top level.
- Add the two keys inside `env`. If an `env` object already exists, add the keys to it; do not replace the object.
- Keep every existing key unchanged.
- Keep the file valid JSON and its permissions private (`0600` on Unix).

What the settings do:

| Key | Value | Effect |
|---|---|---|
| `subagentPromptCacheTtl` | `1h` | Subagents keep a warm prompt cache through a stall instead of re-reading cold after 5 minutes. Cache writes cost more; pays off when workers idle longer than 5 minutes between turns. |
| `env.CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | `5` | The client refuses a sixth concurrent Agent-tool worker. |
| `env.CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` | `4` | Workflow scripts run at most four agents at once. |

Settings are read at session start. Sessions already open keep the old values.

## 3. Verify

- `~/.claude/settings.json` parses as JSON and contains the three keys.
- Either `~/.claude/skills/fan-out/SKILL.md` exists, or `~/.claude/CLAUDE.md` contains the import line and the path it
  names exists.
- Start a new session and ask: "What is the fan-out concurrency rule?" The answer should mention 3-5 concurrent
  streams per wave.

## Uninstall

Remove the skill directory or the import line, and delete the three settings keys. Nothing else was changed.
