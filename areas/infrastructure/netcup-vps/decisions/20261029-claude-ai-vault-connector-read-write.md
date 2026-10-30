---
type: decision
id: ADR-0041
aliases: [ADR-0041]
status: accepted  # 2026-10-30, acceptance passed 2026-10-29 from the phone over the public route
impact: high
provenance: measured
device: netcup-vps
constrains: [mcpread, obsidian-remote-mcp, tailscale-funnel, /srv/vault, /srv/vaults/code]
stale: false
created: 2026-10-29T19:00:00Z
updated: 2026-10-29T19:00:00Z
last_verified: 2026-10-29
commit: pending
supersedes: ADR-0037
superseded_by:
tags: [mcp, claude-ai, ingress, funnel, containment, obsidian]
---

# ADR-0041 — claude.ai vault connector: read/write on the live vault, read-only code

## Summary

The claude.ai connector now reads and writes the live vault at `/srv/vault` and reads
`/home/dev/code` through a read-only bind mount. It is served as the principal `mcpread` by a
locally patched obsidian-remote-mcp v1.5.0 behind Tailscale Funnel on 443. It is fenced twice:
by the kernel (systemd path and IP rules, nftables) and by an app-level access policy. This
supersedes [[20260920-public-mcp-endpoint-claude-ai|ADR-0037]], whose mirrors and inbox no
longer exist after the 2026-10 rebuild and vault merge.

## Context

ADR-0037 gave claude.ai read-only mirrors and a separate write inbox. The
[[20261018-single-operator-rebuild|rebuild]] and the
[[adr-0040-vault-merge-and-agent-council|vault merge]] removed all three: there is one vault,
at `/srv/vault`, carried by Obsidian Sync. The server's git (`/var/lib/vault-git`) is its
source of truth, with an hourly snapshot pushed to GitHub (ADR-0040 §3, amended 2026-10-28).
That makes the live vault recoverable without a mirror. Each chat edit can be undone from
git, so the reason ADR-0037 rejected a write path (silent record rewriting with no way back)
no longer holds in that form.

The endpoint is still the host's only public service, and its callers are remote models whose
input can carry prompt injection.

## Decision

Serve the live vault read/write and the code tree read-only to claude.ai. Do it through
`mcpread` and a local fork of obsidian-remote-mcp, fenced by the kernel and by an app-level
policy, and expose only the MCP and OAuth paths through Funnel on 443.

All of the following was measured 2026-10-28/29 unless marked otherwise.

- **Principal.** `mcpread` (uid 1003): `nologin`, linger, member of `vault`, no sudo, no
  Claude Code login. Its own slice: `MemoryHigh=768M`, `MemoryMax=1G`, both for
  `user-1003.slice` and `mcpread.slice`.
- **Server.** `~mcpread/obsidian-remote-mcp`, branch `local/netcup` = upstream `v1.5.0`
  plus eleven local commits. These are four auth-server hardening commits carried over from
  ADR-0037, the access policy, a dependency update, link and symlink handling, the
  global-limiter fix, caller-safe error text and the YAML message. Each change was
  reviewed adversarially before it went live. Node is Ubuntu's `nodejs` 22.
- **Units.**
  - `mcpread-gateway.service` runs supergateway on 8420 (Streamable HTTP, stateful,
    1 h session timeout) with `vault-mcp.js /srv/vault /srv/vaults/code`.
  - `mcpread-auth.service` runs the OAuth 2.1 auth server on `127.0.0.1:8422`, with
    `ISSUER=https://netcup-vps.tail82bb86.ts.net`.
  - Both have `NODE_ENV=production`, `OnFailure=ntfy-alert@%n.service`,
    `IPAddressDeny=any` and `IPAddressAllow=localhost`.
- **Kernel fence: paths.** `ProtectSystem=strict`, `ProtectHome=tmpfs` and
  `ReadWritePaths=/srv/vault`.
  - Inaccessible: `.claude`, `_generated`, `_meta/scratch`, `_meta/legacy-rules`,
    `/var/lib/vault-git`, `/etc/credstore` and `/root`.
  - Read-only: `arch-thinkpad/state`.
- **Kernel fence: network.** nftables:
  - Output drops by `skuid mcpread` to RFC 1918, 169.254/16, 100.64/10, `fc00::/7` and
    `fe80::/10`.
  - Loopback 8420/8422 is dropped for every uid except root and `mcpread`.
  - An input drop for 8420/8422 after `iif lo accept`. supergateway binds every interface
    and has no `--host` option; a tailnet probe from the laptop was dropped (counter 0 → 15).
- **Code tree.** `srv-vaults-code.mount` bind-mounts `/home/dev/code` on `/srv/vaults/code`
  with `ro,nosuid,nodev`. `mcpread` reads it through an access ACL and a default ACL `u:mcpread:rX`.
- **App-level policy.** `/etc/mcpread/vault-policy.json` (root 0644, `VAULT_POLICY`):
  - Every vault: deny `.git` and `.env*`.
  - `vault`:
    - Deny `.obsidian`, `.claude`, `_generated`, `_meta/scratch` and `_meta/legacy-rules`.
    - Read-only: `CLAUDE.md`, `AGENTS.md`, every `_index.md` and `areas/infrastructure/*/state`.
  - `code`: deny `node_modules` and `dist`; everything else read-only.
  - Disabled tools: `move-directory` and `delete-directory`.
  - The server refuses to start in four cases: the policy cannot be read, it names a vault
    that is not served, it disables an unknown tool, or a served vault has no entry.
  - Links are followed only to allowed targets inside the vault, and never written through.
- **Exposure.** Funnel on 443 mounts only `/mcp`, `/.well-known/oauth-protected-resource`,
  `/.well-known/oauth-authorization-server`, `/register`, `/authorize` and `/token`, each
  proxied to 8422. OAuth 2.1 with DCR and PKCE S256 only. The ADR-0037 hardening stays:
  - Redirect-host allowlist `claude.ai,claude.com`.
  - Login and registration limiters.
  - A client-row cap with a grace period.
  - `trust proxy` loopback.

  The static token is gone. n8n's tailnet-only `serve` on 8443 is unchanged.
- **Login password.** It is in `/root/mcpread/login-password` (root 0600) and KeePassXC
  "netcup vaults MCP login". The scrypt hash is in `~mcpread/.config/obsidian-mcp/password-hash` (0600).
- **Version hold.** `apt-mark hold tailscale` at 1.104.1, so a Funnel or serve change
  cannot alter the exposure unseen.
- **Monitoring.** `mcpread-alive.timer` runs hourly as `mcpread`:
  - Locally, the gateway `/healthz` must return 200 and auth `/mcp` must return 401.
  - Publicly, through Tailscale's ingress address from public DNS, `/mcp` must return 401,
    the OAuth metadata 200, and `/healthz` 404.
  - A failure triggers `OnFailure=ntfy-alert@%n.service`.
- **Kill switch** (admin lane): `tailscale funnel --https=443 off && systemctl stop mcpread-auth mcpread-gateway`.

### Recoverability

v1.5.0 has no pre-write hook, so `vault-snap` cannot run before each chat edit. An edit is
recoverable from the hourly `vault-autocommit` snapshot, and from the nightly restic
backup to B2. A chat edit made and changed again within the same hour cannot be told apart in
git (`inferred`). Obsidian Sync's own version history is the finer net for that case
(`inherited-unverified`).

### Where the code lives

When this work started, the only copy of the four ADR-0037 hardening commits was restic
snapshot `906868b7` (2026-10-18). They were restored from it. Today the eleven commits
exist only in `~mcpread/obsidian-remote-mcp` and in the nightly restic backup of `/home`.
**They must land in infra-tools** as patches with an apply script, before the next upstream
rebase.

## Alternatives Considered

1. **Keep ADR-0037's mirror and inbox pattern.** The mirrors and the inbox repo are gone with
   the rebuild. A mirror of a vault that is now edited on the server would also lag behind the
   live files it copies.
2. **Read-only vault, writes through `_inbox/` only.** Safer, but a phone chat could not
   update a project card or a case log. The kernel and policy fences protect the contracts,
   generated state and agent configuration instead.
3. **Run `vault-snap` before every write.** Not possible without patching every write tool.
   The hourly snapshot was accepted instead (see Recoverability).
4. **Funnel the whole auth server on 443.** One mount is simpler, but `/healthz`, `/` and the
   static files would be public. Path mounts keep everything else at 404 (measured).
5. **Cloudflare Tunnel.** Still the fallback named in ADR-0037. It adds a daemon and a
   credential the host does not otherwise need.

## Consequences

### Positive

- One vault, current to the second, readable and writable from the phone with the laptop off.
- Protected files are protected twice. The agent configuration and generated output are
  invisible to the server process, not just refused by it. The contracts and the
  infrastructure state are read-only. This covers `CLAUDE.md`, `AGENTS.md` and every
  `_index.md` (by the app-level policy only), and `arch-thinkpad/state` (also by the kernel).
- The connector can read the code tree without being able to change it, and never sees
  `.git`, `.env*`, `node_modules` or `dist`.

### Negative

- **Reversal of ADR-0037's rejected write path.** Prompt injection in a chat can now edit or
  move ordinary vault notes. That covers anything not denied or read-only, including admin and
  finance notes. The bounds are the policy, the disabled directory tools, the hourly git
  snapshot and the move to `.trash` on delete.
- **Consent phishing** (accepted residual risk, review 1 [4]). Someone who registers a
  client named "Claude" with a `claude.ai` callback can send Luis a login link. The login
  page shows the client name and callback host, but cannot prove who registered it.
- **Guessing ceiling.** The global login limit is 100 attempts per 15 minutes, which allows
  at most about 9,600 guesses a day from any number of addresses. The password is 32 random
  characters (`measured`: length only). An attacker can hold login or registration at 429
  for 15 minutes; this is availability only, as accepted in ADR-0037.
- **Care load.** Each upstream release needs a rebase of eleven commits, the tests (125),
  and a review if the auth server or the tools changed.

## Revisit when

- Upstream obsidian-remote-mcp ships a pre-write hook. Then run `vault-snap` before each write.
- Someone other than Luis gets access to the connector.
- Tailscale changes Funnel path semantics. That is a reason to lift the hold only after
  re-running the public checks.

## Acceptance

Passed 2026-10-29 from claude.ai over the public route (Luis, phone on mobile data, Tailscale off):
- Stats and reads worked.
- `_generated/.keep` gave EACCES without a host path.
- A `CLAUDE.md` append and a write to the code vault were refused.
- The code listing hides `.git`, `.env` and `node_modules`.

The build, the review rounds and the public checks are in
[[../runbooks/mcp-endpoint|the runbook]]. Accepted 2026-10-30; ADR-0037 now carries
`superseded_by` and a dated line pointing here.
