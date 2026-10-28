---
title: "2026-09-12 — coding rig stage 1 complete: review gate live, template scaffolded"
type: session
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-12
commit: pending
created: 2026-09-12
updated: 2026-09-12
tags: [session, netcup, devenv, claude-code, codex, opencode, git, astro, sandbox]
---

# 2026-09-12 — coding rig stage 1 complete

Stage 1 of the plan finished: deploy keys, the worktree + messaging workflow,
the Codex review gate, the phone path, the laptop alias, and the client site
template — scaffolded by Claude Code, reviewed by Codex, fixed, re-reviewed,
merged by hand, pushed. ADR-0035 written and accepted. **First measured value
of the review gate:** two real defects the implementing agent's own
verification could not catch.

## Morning checks

Timer took its first unattended snapshot at 00:47; `/home/dev/` is in it
(1115 entries); `systemctl --failed` empty. Backups are a running system now.

## 1.5 — deploy keys

Two read-only deploy keys (`gh-infra-docs`, `gh-freelance-practice`), one write
key for `site-template`. `git push --dry-run` from a read-only clone answers
`The key you are authenticating with has been marked as read only` — the
negative test that matters. `vps-trust-direction` closes: the deploy-key pattern
ADR-0027 intended is now the measured norm.

## 1.6 / 1.8 — worktrees, messaging, phone

Two sessions, `claude --worktree alpha --name alpha` and `…beta…`, in tmux
panes (`join-pane -s 2 -h` turns a window into a pane). Alpha committed on
`worktree-alpha` and messaged `@beta`; beta received it, verified the commit
against its own worktree, and — wrongly — flagged `README.md` as deleted in the
main checkout. `git status` said otherwise. **Agent observations are claims.**

Nothing was configured for any of this: cross-session messaging and Remote
Control are on by default in 2.1.269 for a claude.ai login. Both sessions
appeared on the phone under Code. `/list-agents` withholds names while Remote
Control is connected; `--name` is what makes a session addressable.

## 1.7 — the review gate, and what it took

`codex-plugin-cc` needs four things Claude's sandbox denied by design, found
one at a time, each by running it:

1. **Node on the host** — the companion is a Node script. `apt install nodejs`.
2. **Codex visible** — `blockReadsOutsideWorkingDirectories` hides all of
   `$HOME`, so `~/.local/bin/codex` read as "no such file" from inside.
   `allowRead: ["~/.codex", "~/.local/bin"]`. (Restart needed; the block
   setting does not reload live.)
3. **Codex writable** — it keeps sqlite state under `~/.codex`.
   `allowWrite: ["~/.codex"]`. Trade-off accepted and recorded in ADR-0035 §3.
4. **Plugin state writable** — the companion creates job dirs under
   `~/.claude/plugins/data/`, and Claude Code refuses sandboxed writes anywhere
   under `~/.claude/plugins` regardless of `allowWrite` (a built-in guard on
   tool-granting surface). Fix: symlink
   `~/.claude/plugins/data/codex-openai-codex → ~/.local/state/codex-plugin`
   and allow-write the target. `excludedCommands` was the documented
   alternative, but its matching rules are not in the reachable docs and
   excluding `node` wholesale was not acceptable.

Two Claude-side facts learned on the way: denied writes exit 0 and produce no
file; the sandbox masks `.bashrc`, `.gitconfig`, `.idea`, `.vscode`,
`.mcp.json` and `.claude/` **in the working directory** with device nodes, so
they show up as "untracked" from inside and the `.claude/skills` symlink has to
be created by a human.

The plugin runs Codex with no profile argument, so the config default
`reviewer` applies; the reviewer's own output says "the environment is
read-only".

Also installed: `ralph-loop`; skills layout `.agents/skills` +
`.claude/skills → ../.agents/skills`.

## 1.9 — laptop

`Host vps-dev` in `~/.ssh/config`, `alias dev='ssh -t vps-dev tmux new -As main'`.
One word from laptop to session. KeePassXC's lock-on-screen-lock fired twice
today; runbook §0a covers it.

## 1.11 — site-template

Prompted Claude Code with the full scaffold spec (Astro, static, German legal
pages, human-run `deploy.sh`, Lighthouse `check.sh`, `CLAUDE.md` = `AGENTS.md`,
skills). Node was 20, so Astro 5.18 (7.x needs 22.12+); Node 22.23 installed
afterwards from NodeSource and the lockfile rebuilt.

Lighthouse could not run inside the sandbox — Chromium needs a Unix socket —
so `! npm run check` from the human shell: 100/94/100/100 on `/`, all 100 on
the legal pages. The 94 was a `mailto:` anchor with no accessible name, which
turned out to be GFM autolinking splitting a raw HTML anchor.

**The review.** `--base master` had nothing to diff (the repo was born on the
scaffold branch); the empty-tree SHA is a tree, not a commit, and the plugin
**hung ten minutes** on that error instead of exiting. Fix: an empty root
commit, rebased under the scaffold — now the convention for every repo.
`--base HEAD~1` then returned:

- **P1** `deploy.sh`'s `TODO-CLIENT` scan matched its own docs and scripts, so
  every real deployment would have aborted after the placeholders were filled.
- **P2** `check.sh`'s readiness probe would pass against any server already on
  port 4321, so the gate could audit a stale dev server.

Both correctness, both invisible to a build that passes. Claude fixed both on
`fix-review-1` (scan limited to `src/` and `public/`; free port + process
ownership check) plus the mailto; Lighthouse 12×100; second review clean;
`git merge --ff-only` by hand; pushed to `Luis-Gordon/site-template`, private,
marked as a GitHub template repository.

`npm audit` after the Node 22 rebuild: 23 findings incl. 1 critical, to be
checked against `--omit=dev` — a Lighthouse-tree dev dependency is the likely
carrier; the Astro 7 upgrade is its own reviewed task.

## Transport faults today

`sudo -i` from `dev` (no sudo by design — use the `luiswg` window); the
`.claude/skills` symlink invisible to the agent that asked for it; a
`<owner>` placeholder pasted literally; `--keep-last 0`-style guesses twice
(empty-tree SHA). Fewer than yesterday.

## Records touched

- `decisions/20260912-netcup-vps-is-the-coding-host.md` — ADR-0035, accepted.
- ADR-0034 → accepted: the table matches the host (Claude, Codex, OpenCode,
  three deploy keys, restic, ntfy).
- `runbooks/coding-rig-stage-1.md` — open items closed; plugin sandbox
  openings added.
- `backlog/vps-trust-direction.md` → resolved.
- Indices.

## Next

Stage 1 is done. Before stage 2: the Chantal discovery answers → Tech Decision
Record → first client repo from the template. Stage-2 automations start once
that site is in build, per the plan.
