---
title: "Agent council design — claude.ai Projects → Claude Code"
type: design
status: frozen   # 2026-10-20 10:37 — changes only from A1–A10 test results
revision: 4   # r4 2026-10-20 10:26: skill execution classes, no skill model overrides, RO launcher, honest sysadmin guarantee; r3 voice-invocable skills; r2 implementation review; r1 09:48
provenance: inferred
created: 2026-10-20
updated: 2026-10-20
reviewers: [research report 2026-10-19, diminishing-returns pass, ChatGPT roster review 2026-10-20, ChatGPT implementation review 2026-10-20 10:11, Luis]
depends_on: "[[2026-10-20-master-vault-design-v4]]"
tags: [claude-code, agents, skills, council, design]
---

# Agent council design

> [!NOTE] Summary
> 21 claude.ai Projects become **5 boundary agents, 2 workers, 2 rule-only personas, 11 skills**; 2 Projects stay in claude.ai. **Boundary agents** have an empty prompt and no `memory` field, so Claude Code's own system prompt stays intact (documented, ≥ v2.1.281); they carry only tools, model and a fail-closed write-scope hook. **Workers** carry real system prompts, because a delegated sub-agent gets only its own prompt. Stance and facts live in synced nested `CLAUDE.md`. Only workers can be delegated to — enforced by `Agent(...)` permission rules, not by wording. Installed at user scope from one git source with a manifest, never deleting a live definition. Installed at v4 step 9, after the merged vault verifies; requires Claude Code ≥ v2.1.295.

## 1. Principles

1. **A separate definition must buy a boundary or isolation** — tool allowlist, model, write scope, or verbose-output isolation. Topic alone is a `CLAUDE.md` section.
2. **Boundary agents: empty body, no `memory`, no `skills:` preload.** Workers: a prompt body ≤ 60 lines with role, deliverable contract and limits.
3. **Prompt text is not access control.** Four separate layers, each named in the matrix (§5): intended scope (prompt/`CLAUDE.md`) · tool availability (`tools:`) · hook enforcement (fail-closed) · OS permissions (v4 §3b, the backstop).
4. **Delegation is enforced.** `permissions.deny` blocks `Agent(<each boundary agent>)`; agents that may delegate list `Agent(researcher, reviewer)` in `tools:`.
5. **Model and effort follow the decision.** Opus·high only for `sysadmin` and `reviewer`. **Skills never set `model` or `effort`** — an inline override persists for the rest of the turn, so skill matching would silently change the session's model. Model is a session/agent decision; a procedure that needs a different model runs in a worker.
6. **Skills are procedures with an output contract, invoked by meaning, not by slash.** Luis works by voice (Wispr Flow, app dictation), which cannot produce a leading `/`. All skills are model-invocable; descriptions lead with the phrases he actually says. Root `CLAUDE.md`: "Spoken 'slash X', 'use X' or 'run X' means invoke skill X. Name an unused skill only when it would have materially changed the result." This is guidance, not a command parser — A7 measures how reliably it fires. **Selecting a skill is not permission to change state** (§3 execution classes). Not preloaded. Retire a skill when the base model matches it. (r3, 2026-10-20 10:17 — reverses r2's manual-only generators; budget cost of 11 short descriptions accepted, watched via `/context`.)
7. **One authoritative record per entity** (case, client, device, decision). Facts may appear in letters, logs and summaries; only the record is authority. Agent hand-off notes hold open work only.

## 2. Roster

### 2a. Boundary agents (empty prompt; launched by name; not delegatable)

| Agent | Model · effort | `tools` | Vault-edit scope (hook) | Shell policy |
|---|---|---|---|---|
| `sysadmin` | opus · high | inherit + `Agent(researcher, reviewer)` | `areas/infrastructure/**`, `_inbox/**`, `_meta/decisions/**` | Bash allowed; `permissions.ask` on common forms of `systemctl`, `nft`, `ssh`, `rm`, `sudo`, `git push` — **catches the usual spelling, not every route** (`git -C x push`, another interpreter). OS lanes are the containment; firewall, disk and credential changes follow their runbooks, which require Luis's go regardless of command spelling |
| `case-manager` | opus · medium | Read, Grep, Glob, Edit, Write, Skill, WebFetch, `Agent(reviewer)` | `areas/admin/**`, `areas/finance/**`, `_inbox/**` | none |
| `solutions-architect` | opus · medium | Read, Grep, Glob, Edit, Write, Skill, WebSearch, WebFetch, `Agent(researcher, reviewer)` | `areas/freelance/**`, `projects/**`, `_meta/decisions/**`, `_inbox/**` | none — cannot touch `~/code` |
| `socratic-tutor` | sonnet · medium | Read, Grep, Glob | none (no write tools) | none |
| `writing-mentor` | sonnet · medium | Read, Grep, Glob | none (no write tools) | none |

`_attachments/letters/` is **not** writable by any agent: originals are immutable evidence, added by Luis or the phone. Extracted text and annotations go in the correspondence note.

`case-manager` delegates only to `reviewer` (a second look at a draft letter); case content never goes to `researcher`, which has web tools.

### 2b. Workers (delegatable; isolated context; prompt-bearing)

| Worker | Model · effort | `tools` | Prompt body holds | Returns |
|---|---|---|---|---|
| `researcher` | sonnet · high | Read, Grep, Glob, WebSearch, WebFetch | discovery → SIFT source check → GRADE-style confidence → synthesis (from the Research Synthesizer Project, trimmed) | ≤ 60-line evidence matrix: claim · sources · confidence · contested? |
| `reviewer` | opus · high | Read, Grep, Glob | adversarial stance; ranked findings, each with a concrete failure scenario; no fixes | findings list, most severe first |

`reviewer` has **no Bash**. Diffs and logs are passed in by the caller (`git diff > _inbox/review-<slug>.diff` or pasted). A clean-room review (no `CLAUDE.md`) is a one-off: `claude --agents '{"reviewer-cleanroom":{…,"omitClaudeMd":true}}'` — never installed, never routable.

### 2c. Rule-only personas (no agent file)

| Persona | Lives in | Launch | Startup check |
|---|---|---|---|
| Freelance business mentor | `areas/freelance/CLAUDE.md` — stance, six modes, "Next action", push-back list | `cc-biz` | `/context` shows `areas/freelance/CLAUDE.md` at startup |
| Health & fitness coach | `areas/health/CLAUDE.md` (≤ 25 lines) + `areas/health/_status.md` | `cc-health` | same, for `areas/health/` |

Launching *in* the folder matters: a `CLAUDE.md` in the working directory loads at startup; one in a subfolder loads only when a file there is touched.

### 2d. Stay in claude.ai
Music Tutor; Japanese Language Tutor. Profile blocks → `areas/self/music.md`, `areas/self/japanese.md`.

## 3. Skills (11)

All skills: model-invocable, no `model`/`effort` fields, description = 3–5 spoken trigger phrases + one "not for" clause. Each skill's eval holds ~10 should-trigger, ~10 should-not-trigger and ~5 ambiguous/multi-intent dictation-style phrases, scored on whether the **Skill tool was actually called**; descriptions tuned with `skill-creator`.

**Execution classes** — written as the first section of each SKILL.md, because spoken requests mix musing, context and the actual ask:

| Class | Skills | Behaviour |
|---|---|---|
| **A — analyse** | `deadline-audit`, `tech-decision`, `venture-check`, `weak-model-prompt`, `flashcards` | Run on clear intent; return analysis or generated text in the reply. Write a file only if Luis asked for one |
| **B — generate a file** | `prd`, `claude-config`, `wrap` | May be selected automatically; creating or replacing a file follows an explicit ask ("write it", "save it"). Otherwise show the draft and ask where it goes. Overwriting an existing file always asks |
| **C — stage, then promote** | `casework`, `source-intake`, `capture` | Low-impact staging (a new `_inbox/` note, a fleeting capture) proceeds on clear intent. Changing an authoritative record — a case's status, amount, deadline; superseding a note — is shown as a proposed diff and needs "yes" |

**Scope check first:** every class B/C skill begins by checking where it is running — the vault root is found, the target folder is one this session may write (boundary agents: hook; plain sessions: the area matches the cwd's area). Outside that, it degrades to class A and returns text. The "intended for" column below is routing intent; the scope check and hooks are what hold.

| Skill | Class | Invocation | Intended for | Contract |
|---|---|---|---|---|
| `/casework` | C | auto | `case-manager`, plain sessions in `areas/admin` | letter/scan/paste → canonical fields (sender, date, kind, amount, deadline, reference), **each with a source reference**: `letter:<sha256-8 of the attachment>#p<page>` (stable even when the note is edited), `luis:<date>`, or `assumed` → case update + index row → optional draft in Luis's German voice. Only `letter`/`luis` fields may appear in a draft as fact |
| `/deadline-audit` | A | auto | same | Do-this-week ≤ 7 · waiting · stale · deadlines ≤ 30 d; every row cites its case. Scheduled runs are a fixed script, not this skill |
| `/tech-decision` | A | auto | `solutions-architect`, plain sessions | decision record: choice, two rejected, monthly cost, account ownership, legal pages |
| `/venture-check` | A | auto | any | 5-dimension viability, verdict, top-3 risks, next step |
| `/prd` | B | auto | any | `spec.md`, technology-agnostic, Given/When/Then |
| `/claude-config` | B | auto | plain sessions in `~/code/infra-tools` | writes agent/skill/rule files to §4; calls `skill-creator` for evals |
| `/weak-model-prompt` | A | auto | `sysadmin`, plain sessions | weak-model-first prompt + 3–5 test cases |
| `/flashcards` | A | auto | any | Anki-ready cards, ≤ 10 unless asked; Japanese reference; repo schemas stay in `flashcard-tools` |
| `/source-intake` | C | auto | sessions with vault write | **stages** imported material in `_inbox/` with `origin`/`evidence`, effective date and stale-claim flags. Promoting into, or superseding, an authoritative record needs Luis's explicit go |
| `/wrap` | B | auto | any | writes a hand-off note `_inbox/handoff-<date>-<slug>.md` (open work, next action). Writes repo `STATUS.md` **only** in a plain code session whose cwd is the repo. Read-only sessions get the hand-off as text |
| `/capture` | C | auto | sessions with `_inbox/` write | fleeting note into `_inbox/`. Where writes are unavailable (tutors), returns the note as text instead |

Rules, not skills: file placement (root `CLAUDE.md`); MCP tool-description rubric (each MCP repo's `CLAUDE.md`); Claude Code session-diagnosis ladder (`~/.claude/CLAUDE.md`, 3 lines).

## 4. File standard and installation

```
~/code/infra-tools/claude/                 # single source, git; version in VERSION
├── agents/<name>.md
├── skills/<name>/SKILL.md (+ reference/*.md)
├── hooks/scope.py  guard.sh  notify.sh
├── hooks/tests/test_scope.py
├── settings.fragment.json                 # permissions.deny Agent(...), permissions.ask for sysadmin shell
├── MANIFEST                               # every installed path + sha256
└── install.sh
```

**Scope:** agents, skills and hooks install at **user scope** (`~/.claude/agents/`, `~/.claude/skills/`, `~/.claude/hooks/`) for each principal that launches them (server: `luis` and `dev`; laptop: `luiswg`). User scope makes every agent discoverable from any cwd (`cc-leet` runs in `~/code`) and avoids the project-trust gap for frontmatter hooks. Hook commands use absolute paths (`$HOME/.claude/hooks/scope.py`).

**`install.sh` — staged release:** checks `claude --version ≥ 2.1.295`, `python3` present. (1) Build `~/.claude/council/releases/<version>/` and verify it against `MANIFEST`. (2) Run `scope.py` unit tests from that release. (3) Write the merged settings to `settings.json.new` and validate it as JSON with the expected deny/ask keys. (4) Cut over: per item, `~/.claude/agents/<name>.md`, `skills/<name>`, `hooks/<file>` are symlinks into the release, swapped with `ln -sfn` (atomic per link); `settings.json.new` → `settings.json` by `mv`. (5) `install.sh --check`; on any failure, re-point every link and settings to the previous release and exit non-zero. Previous releases are kept; **never deletes or renames an installed agent** — retiring one is a separate, manual step after confirming no saved session uses it; merges `settings.fragment.json` into `~/.claude/settings.json` without dropping existing keys; `chmod 0755` hooks; finishes by running the health check (`install.sh --check`: every manifest file present and matching, every agent loadable, hook tests pass).

**Boundary agent template (empty body):**
```yaml
---
name: case-manager
description: Admin/finance casework session. Launched by name.
model: opus
effort: medium
tools: Read, Grep, Glob, Edit, Write, Skill, WebFetch, Agent(reviewer)
hooks:
  PreToolUse:
    - matcher: "Edit|Write|MultiEdit|NotebookEdit"
      hooks:
        - type: command
          command: "$HOME/.claude/hooks/scope.py areas/admin areas/finance _inbox"
          timeout: 5
          onFailure: block
color: red
---
```

**Worker template:** same frontmatter shape without `hooks`, with a prompt body (role, method, deliverable contract, "never": write files, give fixes, exceed N lines).

**`scope.py`** — a small security component, not a convenience script:
- reads the hook JSON from stdin; malformed JSON, missing `tool_input.file_path`, or an unknown tool → exit 2;
- vault root = nearest ancestor of `$CLAUDE_PROJECT_DIR` containing `.obsidian/` (not the cwd), else exit 2;
- canonicalises the target with `os.path.realpath` on the deepest *existing* ancestor plus the remaining components (handles not-yet-existing files and symlinks); rejects `..` remaining after normalisation;
- allowed iff the canonical path equals or is inside a canonical allowed directory (`os.path.commonpath`, so `areas/admin-old` does not match `areas/admin`);
- always denies `CLAUDE.md`, `AGENTS.md`, `.claude/**`, `.obsidian/**`, `_generated/**`, `**/state/**`;
- prints a one-line reason on stderr; exit 0 allow, exit 2 deny; no network, no writes.
Unit tests: allowed path, traversal, symlink escape, sibling-prefix, wrong vault root, protected file, malformed JSON, missing field. Combined with `onFailure: block`, a missing script, missing Python or a timeout also blocks.

**`settings.fragment.json`:**
```json
{
  "permissions": {
    "deny": ["Agent(sysadmin)", "Agent(case-manager)", "Agent(solutions-architect)",
             "Agent(socratic-tutor)", "Agent(writing-mentor)"]
  }
}
```
Whether this deny also blocks `claude --agent sysadmin` is undocumented → **A6 is a prerequisite for installing this fragment**. No fallback is specified until A6 is measured; if the deny blocks direct launch, the design returns here.

**Resume risk:** if a definition disappears, a resumed session falls back to default tools. Mitigations: the installer never removes agents; `install.sh --check` runs nightly via the existing watchdog and pings ntfy on a missing or modified definition; the two read-only personas launch through a function (§7) that passes an explicit built-in tool allowlist and denies all MCP tools on the command line, so the boundary holds even if the agent file is gone — tested by A8 with the file actually removed.

Root `CLAUDE.md` routing lines:
```
Delegate multi-source questions to researcher and verdicts on finished work to reviewer.
Casework, infrastructure changes and drafts run in the main session, never delegated.
```

## 5. Capability matrix

| Definition | Intended scope | Tools available | Hook-enforced | OS-enforced | External transmission | Unattended |
|---|---|---|---|---|---|---|
| `sysadmin` | infra | all incl. Bash, MCP, web | native vault edits only | luis/dev lanes, v4 §3b tiers | web, MCP, ssh | **never** |
| `case-manager` | admin, finance | read/write, Skill, WebFetch, Agent(reviewer) | vault edits ⊂ admin/finance/_inbox | §3b | WebFetch sends URLs (and anything in them) to the web — never put case identifiers in a URL | no |
| `solutions-architect` | freelance, projects | read/write, web, Agent(researcher, reviewer) | vault edits ⊂ freelance/projects/_meta/decisions/_inbox | §3b; no shell | web search queries | no |
| `socratic-tutor`, `writing-mentor` | what Luis opens | Read, Grep, Glob | n/a (no write tools; aliases also disallow) | §3b | none | no |
| `researcher` | the question | read + web | n/a (no write tools) | §3b — can read the whole vault; it is **not** given personal records by callers (routing rule) | web search queries | inside a session |
| `reviewer` | the supplied material | read only | n/a | §3b | none | inside a session |
| scheduled jobs | per job | fixed script | n/a | runs as luis, writes `_generated/` only | ntfy only | yes — **never an agent** |

"Sending, paying, publishing, pushing" is policy plus `permissions.ask` for `sysadmin`'s shell; no other definition has a tool that can do it. Model calls themselves send the session's context to Anthropic — same as the claude.ai Projects did.

## 6. Evaluation and retirement

`_meta/evals/<definition>.md`: five representative tasks each, plus a plain-Claude baseline in the same folder. Each task run **three times**. Rubric: correct · sources cited · scope respected · concrete next action · tokens/latency. **Hard gates override averages:** any attempted write outside scope, any fabricated date/amount/reference, or any action needing a go without one = fail regardless of score. Workers additionally: picked by routing when expected, not picked when not, main-thread context stays small. Retire a definition that loses to baseline across two eval rounds with no hard-gate advantage; retire a skill the base model matches (`skill-creator` A/B). Monthly, **in a server terminal** (`/skill-doctor` usage reports are not available over Remote Control): `/doctor prompt-audit`, `/skill-doctor`, `/context` Skills row. The listing budget is shared with built-in, plugin and claude.ai-synced skills — count those too.

## 7. Aliases

Generated by `install.sh` per host into `~/.config/zsh/council.zsh` with the real vault path (`/srv/vault` on the server, `~/dev/neural-network` on the laptop):

```bash
V=/srv/vault   # laptop: $HOME/dev/neural-network
alias cc-sys="cd $V && claude --agent sysadmin"
alias cc-case="cd $V/areas/admin && claude --agent case-manager --remote-control"
alias cc-arch="cd $V/areas/freelance && claude --agent solutions-architect"
alias cc-biz="cd $V/areas/freelance && claude --remote-control"
alias cc-health="cd $V/areas/health && claude --remote-control"
# read-only launcher: explicit allowlist + MCP denied; exact flag syntax fixed by test A8
cc_ro() { local dir=$1 agent=$2; shift 2
  cd "$dir" && claude --agent "$agent" --tools Read --tools Grep --tools Glob \
    --disallowedTools 'mcp__*' "$@"; }
alias cc-leet='cc_ro ~/code socratic-tutor'
alias cc-write="cc_ro $V/areas/writing writing-mentor --remote-control"
```

## 8. Acceptance suite (v4 step 9; every item must pass on both hosts)

| # | Area | Must hold |
|---|---|---|
| A1 | Agent load | `/context` in `cc-sys` shows the default Claude Code system prompt sections, matching a plain session; worker prompt loads when `researcher` is spawned; `install.sh --check` fails if any agent file is missing |
| A2 | Trust | agents/hooks at user scope load in an untrusted project folder; a project-scope agent file (none should exist) is reported by `--check` |
| A3 | Hook failure | with `scope.py` renamed, Python absent from PATH, malformed JSON injected, or `timeout` exceeded → `case-manager` write **blocked** |
| A4 | Path scope | allowed write succeeds; `../`, symlink escape, `areas/admin-old/`, wrong vault root, `CLAUDE.md`, `_generated/` → blocked (unit tests + one live try each) |
| A5 | Tool boundary | tutors cannot write (tool absent + alias deny); `case-manager` has no Bash; `reviewer` cannot run commands |
| A6 | Delegation | a plain session cannot spawn any boundary agent; `cc-case` can spawn `reviewer` but not `researcher`; `claude --agent sysadmin` still launches with the deny in place |
| A7 | Skill load | the eval set (§3) is run and the Skill tool call is checked in the transcript, including ambiguous and multi-intent phrases; spoken "slash casework" invokes it in ≥ 9/10 tries; no skill is preloaded; a class C skill shows a diff before changing a case; the session model is unchanged after any skill runs |
| A8 | Install/resume | re-running `install.sh` changes nothing; a forced failure at cut-over rolls back links and settings to the previous release; **with `writing-mentor.md` actually removed**, a resumed `cc-write` session still has only Read/Grep/Glob and no MCP tools (then the file is restored); the `cc_ro` flag syntax is confirmed by listing the effective tools |
| A9 | Evidence integrity | `/casework` on a test letter leaves `_attachments/letters/` byte-identical; every extracted field carries a source; an `assumed` amount never appears in the draft as fact |
| A10 | Phone | each pre-created RC session opened from the Code tab shows the right agent, cwd, nested `CLAUDE.md`, and denies an out-of-scope write |

Baseline evals (§6) are recorded after A1–A10 pass. Only then RC units are enabled and `areas/infrastructure/netcup-vps/runbooks/operator.md` documents the aliases.

## 9. Mapping of the 21 Projects (final)

| Project | Becomes |
|---|---|
| Personal Systems Administrator | `sysadmin` agent + `areas/infrastructure/CLAUDE.md` |
| Finance & Legal Case Manager | `case-manager` agent + `/casework`, `/deadline-audit` + admin/finance rules in root `CLAUDE.md` |
| Freelance Business Mentor | `areas/freelance/CLAUDE.md` (rule persona) + `/venture-check` |
| Freelance Solutions Architect + Development Project Architect | `solutions-architect` agent + `/tech-decision` |
| Health & Fitness Coach | `areas/health/CLAUDE.md` + `_status.md` |
| LeetCode Socratic Advisor | `socratic-tutor` agent (stance in `~/code/CLAUDE.md` section or `areas/self/learning.md`) |
| Writing Mentor | `writing-mentor` agent + `areas/writing/CLAUDE.md` (craft principles) |
| Research Synthesizer | `researcher` worker |
| Claude Code Companion | `reviewer` worker + 3 lines in `~/.claude/CLAUDE.md` |
| Claude Project Architect + CLAUDE.md Generator | `/claude-config` |
| PRD Generator | `/prd` |
| Workflow Prompt Engineer | `/weak-model-prompt` |
| Solo SaaS Strategist | `/venture-check` |
| Flashcard PE + Japanese card format | `/flashcards` (+ repo-level schemas in `flashcard-tools`) |
| MCP Protocol Engineer | rubric in each MCP repo's `CLAUDE.md` |
| n8n Workflow Architect | `areas/infrastructure/netcup-vps/n8n/CLAUDE.md` |
| Obsidian PKM Advisor | root `CLAUDE.md` filing rules |
| Music Tutor, Japanese Tutor | claude.ai + `areas/self/{music,japanese}.md` |

Note for the empty-prompt agents: since the body holds no stance, the tutor's and mentor's stance must live where their sessions start — `~/code/CLAUDE.md` gets a short "Socratic mode" section that applies only when the session agent is `socratic-tutor` (stated in the section), and `areas/writing/CLAUDE.md` holds the mentor's feedback rules.
