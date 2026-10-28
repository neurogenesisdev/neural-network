---
title: Persistent credentials on netcup-vps — the enforceable form of ADR-0021's obligation
type: decision
id: ADR-0034
aliases: [ADR-0034]
status: accepted
impact: high
provenance: inferred
device: netcup-vps
constrains: spec/ssh, spec/backups, devenv
stale: false
created: 2026-09-11T10:00:00Z
updated: 2026-09-20T16:40:00Z
last_verified: 2026-09-12
commit: pending
supersedes:
superseded_by:
tags: [decision, credentials, trust-model, agents, backups, devenv]
---

# ADR-0034 — Persistent credentials on netcup-vps

- **Status:** accepted 2026-09-12 — the table below was checked against the host after
  stage 0 and stage 1 of `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`.
  One row changed on measurement: Claude Code's sandbox can read and write `~/.codex`
  (needed by the review plugin) — see [[20260912-netcup-vps-is-the-coding-host|ADR-0035]] §3
- **Provenance:** inferred (design). No credential listed here exists on the host yet
  except the SSH host keys and the Tailscale node key.
- **Amended 2026-09-12 (2.0):** `auto`'s timers run as system units with `User=auto`, so
  API keys for `auto` take ADR-0021's root mechanism (`/etc/credstore` + `LoadCredential=`)
  instead of resting in `auto`'s home — row updated below
- **Amends** [[20260830-no-luks-on-always-online-vps|ADR-0021]] (its negative
  consequence "no long-lived secret in plaintext on disk"). Does **not** touch
  [[20260830-vps-holds-no-credentials-upward|ADR-0027]], which continues to govern
  direction.

## Summary

ADR-0021's residual obligation cannot be met by a machine that runs coding agents
and automations unattended, so it is replaced by five properties every persistent
credential on this host must have — and a table that names each one.

## Context

ADR-0021 rejected full-disk encryption and left one obligation: no long-lived
secret reaches disk in plaintext. That was written for a host expected to run a
handful of containers whose secrets go through `podman secret` or
`LoadCredential=`.

The host is now becoming the primary coding workstation and, later, the automation
runner (plan D1, D7). That brings OAuth refresh tokens for two vendors, API keys
for three, a backup key, per-repository deploy keys, a deployment token for a
client's hosting account, and an alert token. Some of these are read by the
tools that own them from files the tools choose; none can be made to consult a
KeePassXC database that is not here. An unattended, unencrypted server always
holds *some* recoverable bootstrap secret. The obligation as written is therefore
not enforceable, and an unenforceable rule is worse than a narrow one: it gets
violated silently, which is exactly what ADR-0021 was written to prevent.

## Decision

A credential may persist on netcup-vps only if all five hold:

1. **Downward.** It authenticates to something below this host in the ADR-0027
   trust order (a vendor API, GitHub, B2, a client's hosting account). Never to
   arch-thinkpad or pixel-graphene.
2. **Revocable independently.** It can be revoked at the issuer without
   affecting any other credential, and the revocation URL is recorded.
3. **Least privilege, one purpose.** Scoped to one repository, one bucket, one
   project, one topic. A credential that could serve two purposes is two
   credentials.
4. **One owner.** Owned by exactly one Unix principal (`root`, `dev`, `auto`,
   `n8n`, `hermes`, `mcpread`), mode 0600 or 0700, and — where the tool that reads it supports it —
   additionally unreadable by that principal's own agent processes
   (Claude `sandbox.credentials`, Codex permission profile, OpenCode's
   container mount set).
5. **Listed.** It appears in the table below by name, principal, scope and
   revocation path. No values. A credential not in the table is a defect.

Root-owned service secrets keep ADR-0021's mechanism: `LoadCredential=` from a
root 0600 file under `/etc/credstore/`, never an `EnvironmentFile=` or a value in
a unit. The honest statement of what that buys on an unencrypted disk: it keeps
the secret out of the unit's environment and out of `systemctl show`, not out of
root's reach.

## The table

| Credential | Principal | Location (no values) | Scope | Revoke at | Stage |
|---|---|---|---|---|---|
| SSH host keys | root | `/etc/ssh/ssh_host_*` | this host's identity | regenerate + update `known_hosts` on clients | exists |
| Tailscale node key | root | `/var/lib/tailscale/` | tailnet membership | admin console → machine → remove | exists |
| restic repository password | root | `/etc/credstore/restic-password` (0600) | one repository | rotate with `restic key add/remove` from the laptop | 0.2 ✓ |
| B2 application key `netcup-vps-restic` | root | `/etc/credstore/restic-b2-key-id`, `restic-b2-key` | bucket `netcup-vps-restic-k7q2`; exactly `listBuckets, listFiles, readFiles, writeFiles` (**measured 2026-09-11** via `b2 key list --long`). Can hide files (soft delete), cannot hard-delete versions or change the 30-day lifecycle | B2 console → App Keys → delete, or `b2 key delete` with the master key | 0.2 ✓ |
| ntfy topic | root | `/etc/credstore/ntfy-topic` | one topic on ntfy.sh, no account; the topic name is the whole secret | change topic, re-subscribe phone | 0.2 ✓ |
| Claude Code OAuth | dev | `~dev/.claude/.credentials.json` | Max plan, this machine | claude.ai → Settings → sessions/devices; `claude /logout` | 1.3a ✓ |
| Codex OAuth | dev | `~dev/.codex/auth.json` | ChatGPT plan, this machine | chatgpt.com → Settings → Security → log out devices | 1.3b ✓ |
| OpenCode provider credentials | dev | `~dev/.local/state/devbox/opencode/` (mounted into the container only; denied to Claude's and Codex's sandboxes) | OpenAI via ChatGPT login; DeepSeek API key (prepaid balance is the cap). No Anthropic key by design (D2) | OpenAI: log out devices; DeepSeek: console → delete key | 1.3c ✓ |
| GitHub deploy keys | dev | `~dev/.ssh/gh-<repo>` (one per repo) | `gh-infra-docs`, `gh-freelance-practice` read-only; `gh-site-template` write; `gh-neurogenesis-dev` write (added 2026-09-14, key file and alias renamed with the repo 2026-09-15) | repo → Settings → Deploy keys | 1.5 ✓ |
| Cloudflare API token (per client) | dev | `~dev/.config/secrets/<client>.env` (0600; sandbox-denied) | one Pages project | client's Cloudflare dashboard → API tokens (client-owned; rotated at handover) | 1.11 |
| DeepSeek API key `netcup-vps-auto` | root → auto | `/etc/credstore/deepseek-auto-key` (0600) via `LoadCredential=deepseek-key` into `auto-digest.service` (**measured 2026-09-13:** `auto` cannot read the file; the unit fails to start without it and `OnFailure` fires) | one key, prepaid balance is the cap; pattern for every further `auto` vendor key: `/etc/credstore/<vendor>-auto-key`, never `~auto` | platform.deepseek.com → API keys → delete | 2.2 ✓ |
| ntfy topic (reuse) | root → auto | same file, `LoadCredential=ntfy-topic` into `auto-digest.service` | the digest one-liner; same purpose as the alerts (reach the phone) | as above | 2.2 ✓ |
| `auto` deploy key | auto | `~auto/.ssh/gh-automation-inbox` (0600), alias `Host gh-automation-inbox` | `freelance-automation-inbox`, write — **no key to any vault repo** (**measured 2026-09-12:** `ls-remote` on `freelance-practice` → `Repository not found`) | repo → Settings → Deploy keys | 2.0 ✓ |
| healthchecks.io ping URLs | root → auto | `/etc/credstore/hc-<unit>` (0600) via `LoadCredential=hc-ping` into that unit only (`hc-auto-inbox-selftest`, `hc-auto-digest`, `hc-auto-triage`, `hc-hermes-inbox-push`, `hc-hermes-alive`, `hc-n8n-db-dump`, `hc-n8n-alive`, `hc-mcpread-inbox-push`, `hc-mcpread-alive`) | one check each; the URL only lets someone ping, not read | healthchecks.io → check → change ping URL / delete check | 2.1 ✓ |
| Anthropic API key (workspace `netcup-vps-auto`) | root → auto | `/etc/credstore/anthropic-auto-key` (0600) via `LoadCredential=anthropic-key` into `auto-triage.service` (**measured 2026-09-13:** unit fails at credential load without it, `OnFailure` fires) | one Console workspace, $5/month spend limit; read-only tools only (`--allowedTools Read Glob Grep`) | console.anthropic.com → workspace → API keys → delete | 2.3 ✓ |
| GitHub deploy key `netcup-vps-root-ro-mirror` | root | `/root/.ssh/gh-freelance-practice-mirror`, alias `gh-freelance-practice-mirror` | `freelance-practice`, **read-only**; feeds `/srv/vaults/` for `auto`/`hermes` readers | repo → Settings → Deploy keys (or `gh repo deploy-key delete`) | 2.3 ✓ |
| GitHub deploy key `netcup-vps root - read-only mirror - infra-docs` | root | `/root/.ssh/gh-infra-docs-mirror`, alias `gh-infra-docs-mirror` | `infra-docs`, **read-only**; feeds `/srv/vaults/infra-docs` for `mcpread` (**measured 2026-09-20:** `ls-remote` on the inbox repo → `Repository not found`) | repo → Settings → Deploy keys | MCP 1 ✓ |
| GitHub deploy key `netcup-vps mcpread - write - freelance-automation-inbox` | mcpread | `~mcpread/.ssh/gh-automation-inbox` (0600), alias `Host gh-automation-inbox` | `freelance-automation-inbox`, write — the endpoint's only write target (`inbox/chat/`); **no key to any vault repo** (**measured 2026-09-20:** `ls-remote` on `freelance-practice` → `Repository not found`) | repo → Deploy keys / `gh repo deploy-key delete` | MCP 1 ✓ |
| OAuth password hash + static bearer token (obsidian-remote-mcp) | mcpread | `~mcpread/.config/obsidian-mcp/` (0600; exclude from restic as credentials) | one endpoint; `password-hash` (scrypt) is the OAuth login, `token` is for laptop Claude Code only; `oauth.db` holds hashed client secrets/tokens | `node set-password.js` / `openssl rand -hex 32 > token`, restart `mcpread-auth` | MCP 2 ✓ |
| OAuth client registration for claude.ai (DCR) | mcpread | row in `oauth.db` (`clients`), created automatically at connect; access/refresh tokens hashed in the same db | one client per connect; redirect hosts limited to `claude.ai`/`claude.com` by the local hardening | claude.ai → Settings → Connectors → remove; or delete the row / the db and reconnect | MCP 3 ✓ |
| Telegram bot token (`Hermes Secretary`) | hermes | `~hermes/state/.env` (0600; mounted into the container as `/opt/data/.env`) | one bot, one allowlisted user ID | @BotFather → `/revoke` | 2.8 ✓ |
| DeepSeek API key `netcup-vps-hermes` | hermes | same `.env` | secretary chat; account balance is the cap | platform.deepseek.com → delete key | 2.8 ✓ |
| GitHub deploy key `netcup-vps-hermes-write` | hermes | `~hermes/.ssh/gh-automation-inbox` — **host only, never mounted into the container** | `freelance-automation-inbox`, write; used by `hermes-inbox-push.timer` | repo → Deploy keys / `gh repo deploy-key delete` | 2.8 ✓ |
| n8n encryption key, DB password | n8n | podman secrets `n8n-encryption-key`, `n8n-db-password` (file driver under `~n8n/.local/share/containers/storage/secrets`, excluded from restic); encryption key also in KeePassXC | one instance; owner login `luiswg@neurogenesis.dev` in KeePassXC | rotate per n8n docs; `podman secret rm/create` + restart | 2.6 ✓ |
| Gmail OAuth2 grant (`Gmail account`) | n8n | inside n8n's Postgres, encrypted with the n8n encryption key; client ID/secret also in KeePassXC | Google Cloud project `n8n-neurogenesis`, OAuth client `n8n-neurogenesis` (redirect URIs for both the old `:443` and, since 2026-09-20, the `:8443` editor URL), scope `gmail.modify` on the hub Gmail account only (label, draft; the workflow never sends or deletes); consent screen published, unverified | myaccount.google.com → Security → Third-party access → remove; or delete the OAuth client in the Cloud console | 2.9 ✓ |
| Anthropic API key (workspace `netcup-vps-n8n`) | n8n | n8n Header Auth credential `Anthropic n8n`, same encrypted store | one Console workspace, $5/month spend limit; Haiku classification calls only | console.anthropic.com → workspace → API keys → delete | 2.9 ✓ |
| ntfy topic (reuse) | n8n | `~n8n/n8n.env` (0600) as `NTFY_TOPIC`, read via `$env` (`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) | classifier + error-workflow pushes; same purpose (reach the phone). Deviation from the "never an `EnvironmentFile=`" rule above, accepted for a topic name: podman secrets are not reachable from n8n expressions and n8n Variables are a paid feature | as above | 2.9 ✓ |

Not on this host, by design: the B2 **master** key and `laptop-vps-restic-admin`
(full rights on the bucket) live only in KeePassXC and are used from arch-thinkpad
for key management, retention (`forget --prune`) and restores.

Forbidden here, by construction: any key in `arch-thinkpad`'s or
`pixel-graphene`'s `authorized_keys`; a `gh` CLI token (a copyable file with
account-wide reach — ADR-0021's own example); a Cloudflare token scoped to an
account rather than a project; a shared "utility" key used by two principals.

## Alternatives Considered

1. **Keep ADR-0021's obligation literally and run agents elsewhere** — rejected
   by plan D1: the operator chose sessions that survive the laptop, and no other
   host exists.
2. **LUKS after all** — rejected by ADR-0021 itself; the threat model has not
   changed, only the number of secrets.
3. **A secrets manager on the host (Vaultwarden, `pass`, sops)** — rejected: the
   unlock secret for the manager lands on the same disk, so it adds a layer
   without adding a boundary, and the harnesses cannot read from it anyway.
4. **`systemd-creds encrypt` for root secrets** — considered; on a VM without a
   TPM the sealing key is `/var/lib/systemd/credential.secret`, another
   root-readable file. Same trade as `LoadCredential=`. Not adopted now; it can be
   added per secret later without changing the table.

## Consequences

### Positive

- The rule can be checked: `find / -perm /077 -path '*credstore*'`, `ls -la` of
  the listed paths, and a diff of the table against the host — all mechanical.
- A compromise is bounded by scope: one bucket, one repo, one Pages project, one
  provider key with a spend cap.
- Revocation is a known list of URLs, not an investigation.

### Negative

- **Anyone with root on this host holds every credential in the table.** That is
  ADR-0021's accepted consequence restated, now with a longer list.
- The OAuth tokens are the two that ADR-0021's mechanisms cannot reach: the
  vendors' CLIs choose the file and the format. Mitigation is property 4's
  second half (the agent cannot read its own token file from a sandboxed
  command) and revocation from the vendor side.
- The table is a maintenance obligation. It goes stale the first time a
  credential is added without editing this record. The netcup collector
  ([[../backlog/netcup-collector|backlog]]) should eventually emit the
  path/owner/mode half of it.

## Revisit when

A credential is needed that fails one of the five properties, or the netcup
collector exists and can generate the table's mechanical half.

## Related

- [[20260830-no-luks-on-always-online-vps|ADR-0021]] — amended by this record
- [[20260830-vps-holds-no-credentials-upward|ADR-0027]] — direction; unchanged
- [[../../arch-thinkpad/decisions/20260825-harness-state-and-identity|ADR-0017]] — the laptop's split of harness config, state, credentials and cache; the backup exclusions here mirror it
- [[../../arch-thinkpad/decisions/20260903-agent-containment-two-lanes|ADR-0033]] — two lanes; this host's coding lane per the plan
- `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md` — the plan this record implements step 0.1 of
