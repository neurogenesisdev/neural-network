---
title: netcup-vps is the coding host — three harnesses, three boundaries, humans push
type: decision
id: ADR-0035
aliases: [ADR-0035]
status: accepted
impact: high
provenance: measured
device: netcup-vps
constrains: devenv, spec/ssh, spec/backups
stale: false
created: 2026-09-12T13:00:00Z
updated: 2026-09-12T13:00:00Z
last_verified: 2026-09-12
commit: pending
supersedes:
superseded_by:
tags: [decision, devenv, agents, containment, claude-code, codex, opencode, git, sandbox]
---

# ADR-0035 — netcup-vps is the coding host

- **Status:** accepted 2026-09-12 — built and measured over 2026-09-11/12; every
  claim below was observed on the host
- **Provenance:** measured
- **Extends** [[../../arch-thinkpad/decisions/20260903-agent-containment-two-lanes|ADR-0033]]
  (two lanes) to this device; implements
  [[20260911-persistent-credentials-on-netcup-vps|ADR-0034]]'s table for the
  coding principal

## Summary

Coding happens on netcup-vps as user `dev`, in tmux, with three harnesses that
each get the strongest boundary they support: Claude Code on the host under its
OS sandbox with credential denies; Codex on the host under permission profiles;
OpenCode in a rootless container with only the worktree mounted. Agents commit
on branches; Codex reviews; a human merges, pushes and deploys. The laptop and
phone are thin clients.

## Context

The plan in `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`
chose the VPS as the one authoritative development host (always on, no
inbound-routing problem at home, sessions survive the laptop) and rejected a
model router in front of Claude Code, third-party orchestrators, and running
Claude Max credentials in any non-Anthropic harness (vendor terms). Two rounds
of second-model review moved the containment design from "a Unix user + 0600"
to per-harness enforcement; that reasoning is recorded in the plan's revision
notes and not repeated here.

## Decision

1. **Principals.** `dev` codes (no sudo; SSH login; tmux; rootless podman;
   `MemoryMax=5G`). `auto` will run timers (no SSH login). `luiswg` + `sudo -i`
   administers. `root` stays bash — recovery paths assume it.
2. **Egress.** nftables `output` chain drops new connections from `dev`/`auto`
   to RFC 1918, 169.254/16 and 100.64/10; internet and loopback open. Measured
   to cover rootless-container traffic too.
3. **Claude Code** on the host: `sandbox.enabled`, `failIfUnavailable`,
   `allowUnsandboxedCommands: false`; `sandbox.credentials` deny on `~/.ssh`,
   its own `.credentials.json`, `~/.config/secrets`, `~/.local/state/devbox`;
   `blockReadsOutsideWorkingDirectories`; context vaults via
   `additionalDirectories` + `denyWrite`. **Two deliberate openings, measured
   necessary for the Codex review plugin:** `allowRead` + `allowWrite` on
   `~/.codex` (Codex keeps sqlite state there; its binary lives there), and
   `~/.claude/plugins/data/codex-openai-codex` symlinked to a writable
   `~/.local/state/codex-plugin` because Claude Code refuses sandboxed writes
   under `~/.claude/plugins` and `allowWrite` cannot override that. Trade-off
   accepted: Claude's sandboxed commands can read Codex's ChatGPT token and
   write Codex's config — both are the operator's own, revocable; the review
   gate is worth more than the symmetry. The clean variant (a second
   `CODEX_HOME` for the plugin) is stage 3.
4. **Codex** on the host: `default_permissions = "reviewer"` (`:read-only`,
   secrets denied, no network), `builder` (`:workspace`, `glob_scan_max_depth`
   set because Linux bwrap has no native `**`). Deny `~/.codex/auth.json`, not
   `~/.codex` — the binary is there. The plugin does not name a profile; the
   config default applies (measured: the reviewer reports "environment is
   read-only").
5. **OpenCode** in a Debian container as uid 1001, two mounts (worktree, own
   state), `--cap-drop=ALL`, no host home. OpenAI via ChatGPT login, DeepSeek
   via key; **no Anthropic key** — Claude models run in Claude Code on the Max
   plan.
6. **Git.** Every repo starts with an empty root commit (so a whole-tree review
   has a base). Agents commit on branches. `/codex:review --base master` before
   any merge. A human runs `git merge --ff-only`, `git push`, `deploy.sh` —
   from a plain shell or the `!` prefix. Per-repo deploy keys: read-only for
   context vaults, write for repos `dev` owns; never a laptop key.
7. **Reviews of agent claims are claims.** Two were wrong on the first day
   (a "deleted README" that was not; an empty-tree SHA offered as a base).
   Verify with `git status`, not by trust.

## Alternatives Considered

1. **One `dev` user + 0600 as the boundary** — rejected: a Unix user does
   nothing against an agent running *as* that user.
2. **A separate Unix user for OpenCode** — rejected: shared-worktree ownership
   friction (groups/ACLs on every repo) for the same result the container gives.
3. **Containers for every harness** — rejected: Claude's cross-session
   messaging registers sessions in files under the user's home; the docs state a
   session in a container and one on the host cannot reach each other, and peer
   messaging between terminals was a stated requirement.
4. **Agents allowed to push/deploy** — rejected: the review gate is a merge
   gate; a coder holding the key to bypass it makes the gate decorative.
   Autonomous deploys later = `sandbox.credentials` `mask` scoped to the
   deploy API, never a weakened sandbox.
5. **Per-user Node version manager** — rejected for now: installs under
   `~/.local/share`, invisible to the sandbox without yet another `allowRead`.
   System Node 22 (NodeSource) instead; per-project pinning when two clients
   need different majors.

## Consequences

### Positive

- Measured on day one: a reviewer from another model family found two real
  defects (a deploy guard that always aborted; a Lighthouse gate that could
  audit the wrong server) that the implementing agent's own verification could
  not — both only bite later.
- Laptop failure costs nothing; phone reaches every session via Remote Control
  (on by default for claude.ai logins in 2.1.269).
- Every claimed boundary maps to an enforcement primitive below the thing it
  constrains: kernel sandbox, container mount set, nftables uid match, deploy
  key scope.

### Negative

- **Anyone with root holds every credential** — ADR-0021's accepted
  consequence, now with a longer table (ADR-0034).
- Claude's sandbox can read/write Codex's state (point 3). Named, accepted,
  with a stated cleaner path.
- Lighthouse cannot run inside Claude's sandbox (Chromium needs a Unix socket);
  the check is a human step via `!`. Acceptable: verification, not mutation.
- Permission profiles in Codex are beta; the plugin hangs rather than exits on
  a bad `--base`. Re-check on every upgrade.
- KeePassXC locks on screen lock, so every return to the keyboard begins with
  an unlock before `ssh` works. That is ADR-0004's control doing its job; the
  lockout runbook §0a names the symptom.

## Verification (all run 2026-09-11/12; repeat after any harness upgrade)

```bash
# as dev
nc -zvw3 api.anthropic.com 443; nc -zvw3 <TS_IPV4_arch-thinkpad> 22   # 0 / timeout
opencode-box bash -c 'ls ~/.ssh; timeout 3 bash -c "</dev/tcp/<TS_IPV4_arch-thinkpad>/22" || echo BLOCKED'
# inside claude
run: cat ~/.ssh/authorized_keys          # denied
run: touch ~/x                           # no file appears (verify with ls outside)
/codex:setup                             # ready, logged in
/codex:review --base master              # returns findings or "no actionable"
# inside codex (reviewer): a write must fail with "Read-only file system" even after approval
```

## Revisit when

OpenCode gains peer-session messaging worth having across harnesses (then it
moves to the host under a Claude-grade sandbox or talks via its server API); or
the Codex plugin gains a documented state-dir/`CODEX_HOME` setting (then point 3's
openings narrow); or a task appears that needs both lanes at once.

## Related

- [[20260911-persistent-credentials-on-netcup-vps|ADR-0034]] · [[../../arch-thinkpad/decisions/20260903-agent-containment-two-lanes|ADR-0033]] · [[../../arch-thinkpad/decisions/20260818-devcontainer-agent-exposure|ADR-0003]] / ADR-0004 (KeePassXC timer)
- [[../runbooks/coding-rig-stage-1|runbook: coding rig stage 1]] · [[../sessions/20260912-coding-rig-stage-1-complete|session 2026-09-12]]
- `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`
