---
type: analysis
status: draft
updated: 2026-10-19
tags: [claude-code, agents, migration]
source: _scratch/2026-10-19-claude-projects-export.md
---

# claude.ai Projects → Claude Code council: inventory, grading, disposition

Measured from the raw export (21 projects, 231 KB). Grades are mine; the numbers are not.

## 1. Two generations, not one

| | Gen 1 — Jan 2026 | Gen 2 — Sep/Oct 2026 |
|---|---|---|
| Projects | MCP PE, Writing, PRD, CLAUDE.md Gen, Workflow PE, Japanese, Music, Dev Arch, Flashcard PE, Solo SaaS, Research Synth, Project Architect, Obsidian PKM, n8n WA | Freelance SA, Freelance BM, LeetCode, CC Companion, Sysadmin, Case Manager, Health |
| Template | "Claude Project Architect" shape: Role / Objective / Guidelines / Scope / Constraints / Output / Uncertain / Priority / `<example>` blocks | Hand-written `##` sections, operating modes, handoff boundaries, "Next action", fact discipline |
| Addressee | "the user" (0 mentions of Luis) | Luis, his vault, his hours, his server |
| Size | 9–20 k chars; **32–72 % of the text is few-shot examples** | 2–12 k chars; 0 % examples |
| Vault-aware | No (knowledge-base attachments instead) | Yes (reads `CLAUDE.md`, `_index.md`, writes back) |

Consequence: Gen 2 instructions port almost verbatim. Gen 1 instructions do **not** port — their examples were written for a chat UI, their "user" is generic, and in Claude Code the role/voice goes in a ≤80-line agent body while rubrics and templates go in skill reference files. Treat Gen 1 as raw material, not as agents.

## 2. Per-project grade and disposition

Grade = how much of the current text survives into the new system. **A** port as-is with trims · **B** rewrite, keep the substance · **C** mine one or two rubrics, discard the rest · **D** retire.

Disposition key: **agent** = main-session persona (`claude --agent x`, invoked by name, never auto-routed) · **worker** = sub-agent Claude may delegate to (needs a permission boundary or verbose-isolation reason) · **skill** = `SKILL.md` + reference files · **rules** = nested `CLAUDE.md` / `.claude/rules` · **stay** = remains a claude.ai Project.

| # | Project | Gen | Grade | Stale facts | Disposition |
|---|---|---|---|---|---|
| 1 | Personal Systems Administrator | 2 | A | none (it is this chat's own prompt) | **agent `sysadmin`** — opus/high. Body = current text minus the infra table (that lives in `infrastructure/_index.md`). Also the operator worker for scheduled runs. |
| 2 | Finance & Legal Case Manager | 2 | A | `~/dev/neural-network` path; "netcup MCP" | **agent `case-manager`** — opus/high. Writes only `areas/admin`, `areas/finance`. Skills: `/draft-letter`, `/intake-letter`, `/weekly-triage` (all three procedures are already in the text). |
| 3 | Freelance Business Mentor | 2 | A | Debian, "no hosting stack", `<VAULT_PATH>`, folder list superseded by merge design | **agent `business-mentor`** — opus/high. Six modes stay; vault section replaced by one line pointing at `areas/freelance/CLAUDE.md`. |
| 4 | Freelance Solutions Architect | 2 | A | Debian/"otherwise empty", OpenClaw/Hermes, Claude Desktop MCP | **agent `solutions-architect`** — opus/high. Absorbs #14. Decision rules → `reference/decision-rules.md` in a `/tech-decision` skill. |
| 5 | Health & Fitness Coach | 2 | B | "12-hour security shifts" (job changed Sept 2026), Dihexa/electrolyte state | **agent `health-coach`** — sonnet/medium. The *description* is actually a status note → move it to `areas/health/_status.md` and let the agent read it; keep the 5-paragraph instruction block. |
| 6 | LeetCode Socratic Advisor | 2 | A | none | **agent `socratic-tutor`** — sonnet/medium, `tools: Read, Grep, Glob` (cannot write the solution = enforced persona). Prose style is distinctive; keep it. |
| 7 | Writing Mentor | 1 | B | none | **agent `writing-mentor`** — opus/high, `disallowedTools: Edit, Write` on drafts (feedback, not rewrites). Keep Role/Guidelines/Constraints (~90 lines); craft-principle lists → `reference/craft.md`; drop all 5 examples. |
| 8 | Research Synthesizer | 1 | B | none | **worker `researcher`** — opus/high, verbose-isolation case. Keep the 5-phase workflow + GRADE scale; drop examples. Note: claude.ai Advanced Research still wins for 50-source web dives. |
| 9 | Claude Code Companion | 2 | C | **entirely**: Windows, `local-pc` MCP, `C:\Users\luisw`, flashcard-tools paths | **retire**. Code-review mode → **worker `reviewer`** (read-only, `omitClaudeMd: true`, opus/high) + Codex plugin. Session-diagnosis escalation ladder → 3 lines in `~/.claude/CLAUDE.md`. Config mode → `/new-agent`. |
| 10 | Claude Project Architect | 1 | C | targets claude.ai Projects | **skill `/new-agent`** — discovery questions + the 12-dimension checklist re-targeted at agent/skill frontmatter and the standard in §4 below. Pair with Anthropic's `skill-creator`. |
| 11 | PRD Generator | 1 | A | none | **skill `/prd`** — shortest Gen 1 text, already procedure-shaped. |
| 12 | CLAUDE.md Generator | 1 | B | "~150-200 instruction budget" claim unsourced | **skill `/claude-md`** — keep workflow + validate checklist; drop examples. Used by `solutions-architect`. |
| 13 | Workflow Prompt Engineer | 1 | A | none | **skill `/weak-model-prompt`** — reference.md becomes the skill's reference file. Preloaded by `sysadmin` when touching n8n. |
| 14 | Development Project Architect | 1 | C | Next.js 14, "2025 default", MV2 dates | **merge into #4** as `reference/stack-defaults.md`; refresh versions when first used. |
| 15 | Solo SaaS Strategist | 1 | C | none, but 72 % examples | **skill `/idea-viability`** — the 5-dimension framework + verdict format (≈60 lines). Devil's-advocate second opinion = `reviewer` worker with a different prompt, not a new agent. |
| 16 | Flashcard Prompt Engineer | 1 | B | structured-outputs beta header | **project-level skill** inside the flashcard-tools repo (`.claude/skills/flashcard-prompts/`), not user-level — it is domain knowledge for one codebase. |
| 17 | MCP Protocol Engineer | 1 | C | macOS paths, spec 2025-03-26, Claude Desktop, Anki project | **retire** as persona. Tool-description rubric (3-4 sentence description, every param described, error recoverability) → `skill /mcp-tool-review` (~40 lines). Anthropic's `mcp-builder` skill covers the rest. |
| 18 | n8n Workflow Architect | 1 | D | Hetzner, Docker Compose, UFW, Caddy, GPG/rclone — all replaced | **retire**. Reality is documented in infra-docs (podman Quadlets, nftables, restic). Becomes `infrastructure/netcup/n8n/CLAUDE.md` (≤30 lines) read by `sysadmin`. Its "Always/Never" list is already in #1. |
| 19 | Obsidian PKM Advisor | 1 | D | generic; superseded by the v3 merge design | **retire**. Phase 2 "where does this note go" → **skill `/file-note`** (inbox triage, 30 lines using the vault's own rules). The 5 PKM pitfalls → one line in root `CLAUDE.md`. |
| 20 | Music Tutor | 1 | — | none | **stay** in claude.ai (at the instrument, phone, photos). Student profile block → `areas/self/music.md` so any future agent can read it. |
| 21 | Japanese Language Tutor | 1 | — | none | **stay** for conversation (voice, mobile). Sentence-mining card format → variant inside `/flashcards`; burnout limit (10–15 new cards/day) → `areas/self/japanese.md`. |

## 3. Resulting council (first core)

**7 agents + 2 workers = 9 definitions. 12 skills.** Within the ≤10 / ≤15–20 ceilings from the diminishing-returns pass; only the 2 workers are ever auto-routed, so description overlap cannot bite the personas.

| Kind | Name | Model · effort | Boundary that justifies it |
|---|---|---|---|
| agent | `sysadmin` | opus · high | Bash on the server; hooks guard SSH/firewall/disk |
| agent | `case-manager` | opus · high | writes only `areas/admin`, `areas/finance` |
| agent | `business-mentor` | opus · high | writes only `areas/freelance`; no code |
| agent | `solutions-architect` | opus · high | plans + reviews; `~/code` read, vault `tech/` write |
| agent | `health-coach` | sonnet · medium | `areas/health` only |
| agent | `socratic-tutor` | sonnet · medium | read-only — cannot write the solution |
| agent | `writing-mentor` | opus · high | no Edit/Write — cannot rewrite the draft |
| worker | `researcher` | opus · high | verbose output isolated; returns a reference doc |
| worker | `reviewer` | opus · high | read-only, `omitClaudeMd`, clean-room verdict |

Skills: `/draft-letter` `/intake-letter` `/weekly-triage` `/tech-decision` `/prd` `/claude-md` `/weak-model-prompt` `/idea-viability` `/mcp-tool-review` `/file-note` `/new-agent` `/flashcards` (+ `flashcard-prompts` project-level in its repo).

Second wave, only if the first proves itself after a month: `/log-workout`, `/case-study` (from business-mentor mode 5), a `pkm-steward` agent if `/file-note` turns out to need conversation.

## 4. Standard every definition follows

```
~/.claude/agents/<name>.md           # ≤80 lines body, ≤2-sentence description
~/.claude/skills/<name>/SKILL.md     # procedure ≤100 lines; reference/*.md for rubrics, templates, examples
```

Agent body sections, in this order: **Role & stance** (what it optimizes for, what it is skeptical of) · **Read first** (3–5 vault paths) · **Deliverable contract** (exact shape; ends with `Next action:`) · **Rules** (≤8, each one testable) · **Writes to** (explicit folder list) · **Out of scope → hand-off** (named agent or "human: Steuerberater"). No examples in the body; if an example is load-bearing it goes in a skill reference file.

Description = trigger phrases in Luis's vocabulary + one "not for" clause. Routing table (one line per worker) in root `CLAUDE.md`; personas launched by name via aliases (`cc-case`, `cc-biz`, …).

Shared conventions carried from Gen 2 into every agent body: plain college-grade English with the term in passing; command-first; fact tags `letter/luis/assumed` where facts matter; verification + rollback for anything risky; never credentials in the vault.

## 5. Open calls for Luis

1. `health-coach` as a 7th persona vs. `areas/health/CLAUDE.md` + `/log-metric` and talking to plain Claude there. Persona only earns its slot if you want the "check WHOOP before advising" stance enforced every session.
2. `socratic-tutor` scope: LeetCode only, or also the Japanese/French conversation drills (same read-only boundary, different reference file)?
3. Whether `researcher` is worth defining at all while claude.ai Advanced Research exists and the `deep-research` skill syncs into Claude Code.
