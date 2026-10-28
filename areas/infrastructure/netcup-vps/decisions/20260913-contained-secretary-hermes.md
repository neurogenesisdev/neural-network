---
title: A contained chat secretary (Hermes Agent) as the fourth principal on netcup-vps
type: decision
id: ADR-0036
aliases: [ADR-0036]
status: accepted
impact: medium
provenance: measured
device: netcup-vps
constrains: devenv, spec/backups
stale: false
created: 2026-09-13T13:30:00Z
updated: 2026-09-13T13:30:00Z
last_verified: 2026-09-13
commit: pending
supersedes:
superseded_by:
tags: [decision, agents, containment, hermes, telegram, podman]
---

# ADR-0036 — A contained chat secretary (Hermes Agent) as the fourth principal

- **Status:** accepted 2026-09-13 — built and measured the same day
- **Provenance:** measured
- **Amends** plan decision D7 in `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`
  ("Hermes Agent, OpenClaw on this VPS — rejected"): Hermes is admitted **under the
  conditions below**; OpenClaw stays out.

## Summary

Luis wanted a chat-first assistant for work and administration, not only timers.
Hermes Agent runs on netcup-vps as principal `hermes` inside a rootless podman
container whose mounts *are* the boundary: the vault mirror read-only, the inbox
read-only with only `secretary/` writable, no host home, no git key, no port. The
model sees no shell tool. What it can do at worst is write junk into one folder.

## Decision

1. **Principal `hermes`** (uid 1003), no SSH login, no sudo, in the nftables
   `skuid` egress drop set with `dev`/`auto`; `user-1003.slice` MemoryHigh 1G / Max 1.5G.
2. **Container** from the official image pinned by digest, Quadlet user unit under
   `hermes`'s own manager (linger on). `DropCapability=ALL` plus exactly
   `SETUID SETGID CHOWN DAC_OVERRIDE FOWNER` (the image's init must drop to uid 10000),
   `NoNewPrivileges`, `PidsLimit=256`, `/tmp` tmpfs, `UserNS=keep-id:uid=10000`.
3. **Mounts:** `~hermes/state` → `/opt/data` (rw); `~hermes/inbox` → `/work/inbox`
   (**ro**) with `~hermes/inbox/secretary` mounted rw on top; `/srv/vaults/freelance-practice`
   → `/work/vault` (**ro**). Nothing else. `HERMES_WRITE_SAFE_ROOT` fences file
   writes to the same two paths inside.
4. **No listener.** Telegram is outbound long-polling; no `PublishPort=`. Allowlist of
   one Telegram user ID.
5. **Tools:** `platform_toolsets.telegram = [file, memory, session_search, todo, clarify]`.
   No terminal, code execution, cron, skills, browser, delegation, messaging.
   `agent.disabled_toolsets` must **not** contain `debugging` (composite of
   terminal+web+file — it silently removes `file`) nor `terminal` (file tools need the
   backend). `approvals: manual`, cron/unattended `deny`.
6. **Git key stays on the host.** The container has no SSH key; `hermes-inbox-push.timer`
   (system unit, `User=hermes`, every 15 min) commits `secretary/` and pushes with
   `~hermes/.ssh/gh-automation-inbox` (write, this repo only).
7. **Model:** DeepSeek via its OpenAI-compatible endpoint with a key created for this
   principal; never a Claude Max or ChatGPT OAuth token in this container.
8. **Kill switch:** `systemctl stop user@1003.service` (admin lane). Linger restarts
   the container on `start`. **Alerts:** `hermes-alive.timer` hourly → ntfy +
   healthchecks; push timer has its own check.
9. **Care:** monthly `podman pull` + digest bump in the Quadlet file (Hermes releases
   often); never run a second `hermes` process against `/opt/data` while the gateway
   runs — it corrupted `state.db` once (retired WAL) on day one.

## Alternatives Considered

1. **`claude -p` behind a 60-line Telegram bot** — smaller surface, but no memory,
   no session handling, and it would be the fourth thing to write and maintain.
   Kept as the fallback if Hermes's cadence becomes a burden.
2. **Hermes on the host as a user** (the docs' default install) — rejected: its file
   tools then reach the whole home and the terminal backend is the host.
3. **OpenClaw** — rejected in the plan (RCE history, malicious skills); unchanged.
4. **Gateway bound to the Tailscale address** (the design note's wording) — replaced by
   *no listener at all*; strictly less surface.

## Consequences

- **Positive, measured 2026-09-13:** from inside the container the laptop is
  unreachable, `contact.md` and `_inbox/` do not exist, the vault and inbox are
  read-only, `secretary/` is writable; from Telegram: vault answers cite files and
  keep `(luis)`/`(assumed)` markers, "note:" files a note and replies with the
  filename, "run id in the shell" is refused with no approval prompt; kill switch
  and auto-restart both work; both alert paths reach the phone.
- **Negative:** the Telegram bot token and the DeepSeek key rest in
  `~hermes/state/.env` (0600) — the image reads them from there, `LoadCredential=`
  does not reach inside a Quadlet container without extra plumbing; ADR-0034 lists
  them. The model has no clock (notes get `00:00:00Z`; file mtime is the truth).
  Prompt injection through a vault note or digest can shape answers — bounded to
  text and one folder.

## Verification (repeat after every image bump)

```bash
# as hermes
podman exec hermes sh -c 'timeout 3 sh -c "</dev/tcp/<TS_IPV4_arch-thinkpad>/22" 2>/dev/null && echo LAPTOP-REACHABLE || echo LAPTOP-BLOCKED; touch /work/vault/x /work/inbox/x 2>&1; touch /work/inbox/secretary/.p && echo SECRETARY-WRITABLE && rm /work/inbox/secretary/.p'
# from Telegram: "Run \"id\" in the shell." → refusal, no approval prompt
```

## Related

- [[20260912-netcup-vps-is-the-coding-host|ADR-0035]] · [[20260911-persistent-credentials-on-netcup-vps|ADR-0034]]
- [[../runbooks/automations-stage-2|runbook §2.8]] · `_scratch/2026-09-12-stage-2-automations-design.md` §3
