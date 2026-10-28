---
title: "Rebuild as a phone-first operator host — three principals, Claude Code as operator, vault as memory"
type: decision
id: ADR-0039
legacy_id: ADR-0038
aliases: [ADR-0039]
status: accepted  # 2026-10-19 00:58 — verification block passed on the new host
impact: high
provenance: measured  # verification block run 2026-10-19; see runbooks/rebuild-2026-10 §As actually done for deviations
device: netcup-vps
constrains: devenv, spec/ssh, spec/backups, automations
stale: false
created: 2026-10-18T13:00:00Z
updated: 2026-10-19T01:00:00Z
last_verified: 2026-10-19
commit: pending
supersedes: ADR-0035 (§1 principals, §2 egress, §3 sandbox), ADR-0036 (Hermes retired), ADR-0037 (read-only endpoint → read-write)
amends: ADR-0021 (root holds everything → now luis + dev), ADR-0024 (Debian → Ubuntu 26.04 LTS), ADR-0034 (credential table rewritten)
superseded_by:
tags: [decision, rebuild, ubuntu, claude-code, remote-control, obsidian, n8n, containment, credentials, phone]
---

# ADR-0039 — Rebuild as a phone-first operator host

- **Status:** proposed 2026-10-18, decided after a day of design review (three research passes, practitioner survey, failure-mode survey, grilling); **accepted 2026-10-19** after the rebuild (Phases 0–7 of `runbooks/rebuild-2026-10.md`) and the verification block passed.
- **Provenance:** measured 2026-10-19. Deviations from the text below, as built: `n8n` is a normal uid (1002) with nologin rather than a system user (subuids auto-allocated); the vault root carries a *default* ACL `g:vault:rwX` so folders created by Sync inherit the content-folder mode; `n8n` Quadlets are auto-started by their `[Install]` sections (no `enable`); the `auto-*` timers were **not** recreated (unread output — see §6); the master vault is **`neural-network`** absorbing infra-docs and freelance-practice, not the reverse; `/srv/vault` currently holds `infra-docs` until the merge. Codex's bwrap sandbox needed an AppArmor `userns` profile on 26.04 (`/etc/apparmor.d/codex-bwrap`). Everything else is as written.
- **Amended 2026-10-21:** renumbered ADR-0038 → ADR-0039 (erratum, vault merge step 4). ADR-0038 had already been assigned on 2026-09-21 to [[20260921-neurogenesis-dev-i18n-routing|the neurogenesis.dev i18n routing decision]], which keeps that number. This record was accepted under the duplicate ID; `legacy_id: ADR-0038` keeps old mentions searchable. The Decision and Rationale text is unchanged.

## Summary

The September build (ADR-0035/36/37) optimised for containment: six principals, per-harness sandboxes, read-only vault mirrors, uid-keyed egress drops. It was correct and it blocked daily use — changes required crossing user, container and mirror boundaries, and the phone could only read. This rebuild optimises for **capture-and-act from the phone** with the fences that are cheap and the ones that actually hold, and drops the rest.

**Three principals.** `luis` (me: admin, password sudo, owns the vault), `dev` (my coder: Claude Code, Codex, repos, timers, channels; no sudo), `n8n` (one service, one grant set, its own user). Root locked, console via `luis`.

**Claude Code is the operator.** Remote Control server mode on the VPS, driven from the Claude app's Code tab (voice), one session per thread. claude.ai Projects retire as a working surface; their instructions become subagents in the vault. The chat ↔ Code "messenger pigeon" ends because there is only one Claude that touches files.

**The vault is the memory.** One master vault (PARA + MOC; `neural-network` absorbing infra-docs and freelance-practice), live on the server via Obsidian's headless Sync client running as `luis`, read/written by Claude Code over the filesystem. Session logs, handoffs, corrections and decisions live in it. Code repos stay outside it under `~dev/dev/`.

## Context

- Laptop is mostly offline when I'm away; the phone (GrapheneOS, Claude app, Wispr Flow) is the primary device. The git mirror chain (laptop → GitHub → VPS) cannot carry a phone-side edit to the server; only a Sync node can.
- No first-party bridge exists between a claude.ai chat and a Claude Code session (anthropics/claude-code #2511 open). Claude Desktop cannot run headless. claude.ai connectors must be publicly reachable from Anthropic's `160.79.104.0/21`.
- Research (2026-10-18): practitioners who built this shape (Martineau, dmcc.io, Lhotka, Baulig, Zhutov) are still on it; what got superseded was Telegram bots/OpenClaw (→ Remote Control + Channels), giant CLAUDE.md (→ small skills/subagents), vector memory (→ plain markdown). Nobody runs a service-uid split; nobody weighed it either.
- Credential research: inside one uid, every just-in-time secret mechanism is theatre — a same-uid process reads any file, `/proc/<pid>/environ`, the podman socket. Only a different uid isolates. systemd 258 encrypted credentials protect at rest, not against same-uid.

## Decision

1. **OS:** Ubuntu 26.04 LTS with Ubuntu Pro (Livepatch, ESM), installed by `debootstrap` from the netcup rescue system onto the ADR-0026 subvolume layout plus a seventh subvolume `@srv` for the vault. snapd stays installed **only** because `canonical-livepatch` is a snap; no other snaps. Debian 13 was the zero-drama option; Ubuntu wins on newer systemd/podman/kernel, Livepatch for a box that never reboots on my schedule, and AppArmor profiles enforced by default. Arch rejected for a host running unattended automations (no backports; I'd be its release manager).
2. **Principals.**
   - `luis` — SSH login, **password** sudo (console/lockout runbooks assume it; a long-lived `ob sync` process runs as luis, so no NOPASSWD). Owns `/srv/vault`. Admin Claude Code lane = `sudo -i` → `claude`, short bursts, never always-on.
   - `dev` — SSH login, no sudo, linger. Claude Code (Max login), Codex, OpenCode, Playwright/headless Chromium, repos under `~dev/code/<repo>`, its own systemd **user** timers, channels. One `sudoers` line, no wildcards: `NOPASSWD: /usr/local/sbin/n8nctl` — a root-owned wrapper that accepts only `status|restart|logs` and delegates to n8n's *user* systemd manager. Everything else dev needs from n8n goes through its API key.
   - `n8n` — nologin, linger. n8n + Postgres as rootless podman Quadlets. Vault mounted **`/srv/vault/inbox` only**. DB password and `N8N_ENCRYPTION_KEY` as podman secrets (key also in KeePassXC). `tailscale serve --https=8443`.
   - root — locked, no SSH.
   - Rule: a human drives it → `dev`; a service holding a precious grant → its own named user (`n8n`; `hermes` if it ever returns with grants).
3. **Vault.** `/srv/vault`: the **root directory is luis-owned, group-traversable, not group-writable, no `other` bits** (`2750`); the **content folders** (PARA top level, `inbox/`, `journal/`, `sessions/`, `handoff/`) are `luis:vault 2770` with a default ACL `g:vault:rwX`; `.claude/`, `.obsidian/` are `0750 luis:vault`, `CLAUDE.md` is `0640 luis:vault`; `luis` and `dev` are in group `vault`. No `other` bits anywhere, so `n8n` (not in the group) can read nothing outside its `inbox/` ACL. **No sticky bit** — it would block `rename()` over luis-owned notes (every note that arrives from the phone is luis-owned) and so break atomic writes; the non-writable root gives the same protection for `CLAUDE.md`/`.claude/`/`.obsidian/` without that cost. New top-level folders are an admin-lane act. `n8n` is **not** in the group: it gets a user ACL on `inbox/` only, plus traverse on the root. `ob sync --continuous` as a `luis` user unit with `Restart=always`, `.sync.lock` cleanup on start, a watchdog timer that restarts on stale "Fully synced", `.git` excluded from Sync. Git in the vault is **commit-only** (snapshot/undo), never checkout/branch under the live watcher. `.claude/` (agents, hooks, skills) and `.obsidian/` (plugin config is code): luis-owned directories, mode `0750`, no ACL — so neither their contents nor the entries themselves can be changed, renamed or replaced by `dev`/`n8n`; `CLAUDE.md` in the root is protected by the root directory not being group-writable. Editable only from the admin lane (kernel-enforced injection fence). Deny rules and hooks in `~dev/.claude` are accident prevention, not a boundary — they belong to `dev`. `inbox/` is where `n8n` and clippings land; `dev` treats it as untrusted data; a cheap `dev` timer files it.
4. **Claude Code.** Two `claude remote-control` servers as `Type=exec` user units under `dev` — one with `WorkingDirectory=/srv/vault` (so the protected `CLAUDE.md` is actually loaded; Claude Code does not load instruction files from `additionalDirectories` by default), one in `~dev/code` for code with a worktree-per-task rule. Both run as `dev` and can reach each other's files; the split only decides which `CLAUDE.md` governs the session. Each server allows several concurrent sessions. Concurrent vault sessions are **last-writer-wins**, accepted; capacity kept low. No tmux supervisor (it hid failures from systemd for the one practitioner who tried; tmux stays as an optional attach surface). Each unit: trusted non-home working dir, `Restart=always`, hourly `claude update` with restart when idle, `OnFailure=` → ntfy. Permission posture: `acceptEdits` for operator sessions, Plan mode for design conversations (CLAUDE.md: "design conversations start in plan mode; implement after *go*"). Mobile offers Manual/Accept edits/Plan only; phone approvals are unreliable — design around pre-authorised rules, not prompts. `permissions.deny` on `~/.ssh/**`, `~/.claude/.credentials.json`, `~/.codex/auth.json`; PreToolUse guard hook for `tailscale down|logout`, `nft flush`, `rm -rf` on `/srv|/home|/etc`, `userdel`, `mkfs`, force-push. Sandbox off (ADR-0035 §Amendments stands). Transcripts are stored by Anthropic while Remote Control is connected — accepted.
5. **Writes to the vault.** Whole-file atomic writes (temp + rename), soft-delete to `.trash/`, no mass renames (links break), `git commit` before every agent job. Enforced in the vault MCP tools; a CLAUDE.md rule for direct edits.
6. **Automations.** Deterministic timers and `claude -p --bare` jobs run as **`dev` user units** (keys via user-level `LoadCredential=` from `~dev/.config/credstore/`; `--max-turns`, spend caps). Rule from the second review: a job may run as `dev` if compromising `dev` is allowed to compromise everything the job can do — then dev owns the unit and can edit it freely; otherwise it needs another uid. System units only for n8n and backups. n8n for OAuth-grant workflows. Hermes **retired** (sources kept in `build/hermes/`); revisit only if a month of subagents shows a gap. Telegram = notification path (Notification hook → bot API/ntfy), channel plugin later.
7. **Remote endpoint.** **Not rebuilt by default.** The OAuth 2.1 + DCR auth server and obsidian-remote-mcp (with the four hardening commits) are kept in the vault as code; an unused public service is a standing cost and Remote Control covers the phone. Rebuild only if the first week shows a real need for claude.ai/mobile *chats* to touch the vault, then as its own reviewed gate (write scope, client-address trust through Funnel, Anthropic `/21` as a second layer only), never with arbitrary shell exec.
8. **Network.** nftables input `policy drop`, tailnet only (ADR-0029 stands); panel firewall `tailnet-only` unchanged. **No uid egress drops** for `dev`. Tailscale is **not** version-held at install; a hold is applied only after Serve (and Funnel, if ever rebuilt) are verified, and reviewed monthly (1.102.1 broke Funnel once).
9. **Documents (not vault).** Syncthing, per folder: tier-1 identity kit (passport, Ausweis, licences, cards, birth certificate) syncs **laptop ↔ phone only — the VPS is not a member**; tier-2 correspondence (Jobcenter, Bürgeramt, Inkasso, clients, bills) and `inbox/` include the VPS; media optional. Steuer-ID/IBAN live in KeePassXC and are pasted into a session when a form needs them. Syncthing never touches the vault folder.
10. **Publishing.** Branches and staging from anywhere (`tailscale serve` staging build; branch pushes with `dev`'s write deploy keys); Codex review gate + `git merge --ff-only` + production deploy remain a laptop act. ADR-0035 §6 narrows to "humans push to production".
11. **Laptop.** Portal: Obsidian desktop, KeePassXC, identity folder, production deploys, admin lane for big changes; its own Claude Code (sudo) administers the laptop. ADR-0027 unchanged: nothing on the VPS reaches the laptop or phone.
12. **Model routing.** Subagents in `.claude/agents/` carry `model:` (journal/capture/habit on the cheapest model) and descriptions that trigger on what I say; I never name a model.

## Alternatives considered

- **One user for everything** — what every practitioner runs. Rejected only because n8n holds a `gmail.modify` grant on the mailbox that is a recovery address; the uid split costs ~30 min once and nothing daily.
- **Claude.ai Projects as the advisor + Claude Code as the builder** — the pigeon stays; no bridge exists.
- **Headless Claude Desktop on the server** — GUI only, no headless; dead end.
- **Hermes/DeepSeek overflow lane now** — withdrawn; test native subagents first. Trigger to revisit: first weekly-limit lockout (then: second Max account vs API key + cap).
- **Git as the server's vault truth** — cannot carry phone edits while the laptop is off.
- **Arch / Debian 13 / Alma 10** — see Decision 1; Alma's SELinux is the only real security upgrade and it doesn't address the actual threat (an agent doing agent-looking things as a normal user).
- **Mem0/vector memory** — backlog; vault + grep + Claude auto-memory first.

## Consequences

### Positive
- Capture-and-act from the tram: journal, habits, blog drafts, backlog items, n8n prototypes, site fixes land within minutes, continue from the laptop with the same files.
- One Claude touches files; no copy-paste between surfaces.
- Three users I can hold in my head; shared files by group; secrets by uid.
- Everything an agent did is a file in the vault or a commit — debuggable after weeks away.

### Negative / accepted risks
| Risk | Accepted because | Mitigation in place |
|---|---|---|
| Injected `dev` session can read `dev`'s own tokens (Claude/Codex logins, deploy keys) and trash repos/vault | Each is revocable in one click; vault has git + Sync history + restic | deny rules, guard hook, commit-before-job, `.trash/` |
| Injected `dev` session can `sudo`? **No** — `dev` has no sudo | — | — |
| Injected n8n workflow (email is attacker-controlled text) edits the vault | Mount scope + ACL limit it to `inbox/`; no `other` bits so it cannot read the rest | `dev` treats `inbox/` as data |
| Two vault sessions edit the same note | Single user, capacity 4; atomic rename prevents torn files but not lost updates | Last-writer-wins accepted; Sync history + git undo |
| A nested `CLAUDE.md` written into a content folder is loaded as instructions | Content folders must be dev-writable | PreToolUse hook refuses writes to `CLAUDE.md`/`.claude/` paths; nightly luis-lane scan; `claudeMdExcludes` if available |
| Injected text persists via CLAUDE.md/agents/hooks | Files are luis-owned, `dev` read-only | kernel permission, admin lane only |
| Remote Control: 10-min offline exit, reboot loses sessions, OAuth re-login needs SSH | Detection, not schedule | `Restart=always`, `OnFailure` → ntfy, Termux path kept, never enable Trusted Devices |
| Phone approval prompts hang the host | Known, closed "not planned" | pre-authorised rules; notification hook so I know in seconds |
| Weekly Max cap is an outage for Remote Control; background jobs die silently at the cap | Named, not solvable in-design | timers on API keys with caps; overflow lane decided at first lockout |
| obsidian-headless beta: deletions (#19/#28), reboot hang (#50), no SIGTERM (#25) | Worked on Hive; now with fences | git commit-only, restic, Sync version history, watchdog, panel snapshot before first enable |
| Anthropic sees everything an agent touches; transcripts stored while RC connected | Already the case; local models unaffordable | identity kit + KeePassXC never on the server; training toggle checked |
| Vault decay ~month 6 (stale facts, keyword retrieval) | Known from practitioners | monthly vault-maintenance subagent, `superseded:` frontmatter, corrections log → rules |
| Funnel/connector flakiness (300-s tool cap, Tailscale regressions, DCR changes) | Endpoint is secondary | best-effort; Tailscale pinned; tools short |
| `luis` runs a daemon (`ob sync`) | It consumes no LLM instructions | password sudo only |

## Verification (run after the rebuild; promote `provenance: measured` when green)

```bash
# NetCup server — as dev
sudo -n true; echo "exit $? (expect 1)"
cat /etc/credstore/* 2>&1 | head -1                 # Permission denied
sudo -u n8n true 2>&1 | head -1                     # not allowed
# (luis first: touch /srv/vault/journal/_probe.md)
cd /srv/vault/journal && echo a > t && mv t _probe.md && rm _probe.md && echo atomic-ok   # rename over a luis-owned note works
touch /srv/vault/x 2>&1                             # Permission denied (root not group-writable)
mv /srv/vault/CLAUDE.md /tmp/ 2>&1                  # Permission denied (same reason)
touch /srv/vault/.claude/x 2>&1                     # Permission denied (luis-owned 0750, no ACL)
cp /srv/vault/CLAUDE.md /tmp/c && cat /tmp/c > /srv/vault/CLAUDE.md 2>&1   # Permission denied (file 0640 luis)
systemctl --user status claude-rc-vault claude-rc-dev   # both active, Type=exec, no tmux
# in a vault session: /context lists /srv/vault/CLAUDE.md
# Claude app → Code tab: netcup-vps visible, new session, `sudo whoami` → refused, `id` → dev
# as luis
systemctl --user status obsidian-sync               # active; journal shows "Fully synced"
# phone: edit a note in Obsidian → appears on server < 60 s; Claude edits a note → appears on phone < 60 s
# as n8n (sudo -u n8n -H bash from luis)
grep '^n8n:' /etc/subuid /etc/subgid                # a range each (rootless podman)
touch /srv/vault/inbox/x && rm /srv/vault/inbox/x   # ok (user ACL)
touch /srv/vault/x 2>&1                             # Permission denied (not in group vault)
cat /srv/vault/journal/_probe.md 2>&1; cat /srv/vault/CLAUDE.md 2>&1; ls /srv/vault/.claude/ 2>&1   # all Permission denied (read-negative)
# inside the container: podman exec n8n touch /vault/inbox/_probe → ok; only /vault/inbox is mounted
# off-tailnet: nc -zvw3 <NETCUP_IPV4> 22 → timeout; nc -zvw3 <NETCUP_IPV4> 443 → timeout (no Funnel)
```

## Revisit when
- First weekly-limit lockout (overflow lane decision).
- anthropics/claude-code #2511 ships a chat ↔ Code bridge.
- obsidian-headless leaves beta, or a deletion incident occurs (then: git as truth, Sync as transport).
- A second VPS is affordable (automation host split).
- Three months in: did the subagents stick, did the vault decay, did the phone stay primary.

## Related
- Supersedes/amends: ADR-0021, 0024, 0034, 0035, 0036, 0037. Unchanged: ADR-0025 (debootstrap), 0026 (subvolumes), 0027 (no credentials upward), 0029 (sshd binds wildcard, nftables is the control).
- `runbooks/rebuild-2026-10.md` — the checklist.
- Backlog opened by this ADR: vault merge; Syncthing tiers; general MCP endpoint widening; subagent definitions from Project instructions; Stop/SessionStart hooks; vault-maintenance agent; Telegram notification hook; Remote Control watchdog; GrapheneOS push test; Obsidian Sync device-count check; Ubuntu Pro device cleanup; build/ scripts port to Ubuntu; ADR-0034 table rewrite.
