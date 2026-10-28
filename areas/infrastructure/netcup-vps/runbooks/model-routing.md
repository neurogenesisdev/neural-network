---
title: "Claude Code model routing — subagents by cost"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-17
commit: pending
created: 2026-09-17
updated: 2026-09-24
tags: [runbook, claude-code, subagents, models, quota]
---

# Claude Code model routing (set up 2026-09-17)

> [!NOTE] Summary
> Main session and plan mode run Fable 5.1 (default via `/model`, medium
> effort; arrow keys in the picker raise effort for planning sessions).
> Side work goes to four user-level subagents in `~/.claude/agents/` so
> Fable weekly usage is spent only where it pays. Verified by transcript
> grep: haiku/sonnet/fable all billed as configured.

## Agents (`~/.claude/agents/*.md`, apply in every repo)

| Agent | Model | Use for | Tools |
|---|---|---|---|
| scout | haiku | read-only search, reading, summaries | Read, Glob, Grep |
| builder | sonnet | implementing decided changes, tests, checks | all |
| reviewer | opus | diff review pre-merge, security changes, stubborn bugs | Read, Glob, Grep, Bash |
| deep | fable | only on explicit request, or reviewer failed twice | all |

Routing rule lives in `~/.claude/CLAUDE.md` (§ Model routing). reviewer
does not replace `/codex:review` — the Codex cross-vendor gate still runs
before every merge. reviewer's "do not edit" is instruction, not
enforcement (it has Bash); the real gates are the branch, the human merge,
and the `permissions.deny` rules (the Bash sandbox is off since 2026-09-24 —
[[coding-rig-stage-1]] §Amendments).

`opus` resolves to **`claude-opus-5-5`** (measured 2026-09-23 with
`claude --model opus -p 'Reply with only your exact model id.'`), so reviewer
runs Opus 5.5 without a config change. builder stays on sonnet until one task
is compared against opus (wall time, quality). Lesson 2026-09-21: a
backgrounded builder paused 9 h on a file-delete approval nobody saw — site
work now runs in the foreground, no background subagents.

## Measured 2026-09-17

- `model: fable` is a valid subagent alias (no error, ran).
- Transcript grep over `~/.claude/projects/` (incl. `*/subagents/`):
  haiku 15, sonnet 4, opus 133, fable 584 — routing billed as configured.
- Subagent transcripts live under
  `projects/<session-dir>/<session-id>/subagents/agent-*.jsonl`.
- Background agents do not reliably inherit the working directory: scout
  searched `/home/dev` when the prompt named a relative path. **Always put
  absolute paths in scout/builder prompts.** (The `/agents` wizard is
  removed in 2.1.273; files are the interface.)

## Verify after a Claude Code update

```bash
grep -rhoE '"model":"[a-z0-9.-]+"' ~/.claude/projects/ --include='*.jsonl' | sort | uniq -c
```
New haiku/sonnet entries after delegating = still routing.
