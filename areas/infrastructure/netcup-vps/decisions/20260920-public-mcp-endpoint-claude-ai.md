---
type: decision
id: ADR-0037
aliases: [ADR-0037]
status: superseded  # accepted 2026-09-20 (stage 4 acceptance passed from the phone with the laptop offline); superseded 2026-10-30 by ADR-0041
superseded_by: "[[20261029-claude-ai-vault-connector-read-write]]"
provenance: inferred
device: netcup-vps
created: 2026-09-20
updated: 2026-09-20
tags: [mcp, claude-ai, ingress, funnel, containment, obsidian]
---

# ADR-0037 — Public MCP endpoint for claude.ai (principal `mcpread`)

Amended 2026-10-30: superseded by [[20261029-claude-ai-vault-connector-read-write|ADR-0041]]. The mirrors and the inbox this record describes no longer exist; the connector now reads and writes the live vault.

## Context

Claude chats (web, desktop, and especially the phone app with the laptop off)
need to read the vaults and save notes. The Claude mobile app has no local file
access; claude.ai reaches only **remote MCP connectors**, called from
Anthropic's cloud. So the capability requires a **publicly reachable HTTPS
endpoint** — the first public-facing service on netcup-vps. This amends the
working assumption behind [[20260911-panel-firewall-outer-ring]] ("nothing
public") at the *application* layer only: the panel firewall stays
tailnet-only; ingress arrives over an outbound tunnel.

Spike 2026-09-20: see `../backlog/mcp-read-access-from-desktop.md` for the
findings this decision rests on.

## Decision

- **Principal `mcpread`**: no SSH login, no sudo, `skuid` egress drops (same
  set as `dev`/`auto`), own slice (MemoryHigh 768M / Max 1G).
- **Server**: `rollecode/obsidian-remote-mcp` v1.4.2 — obsidian-mcp (stdio) →
  supergateway (Streamable HTTP :8420, localhost-fenced via
  `IPAddressDeny=any` + `IPAddressAllow=localhost`) → auth-server.js
  (OAuth 2.1: PKCE S256, DCR, refresh rotation, hashed tokens; :8422). Static
  bearer token kept alongside for laptop Claude Code.
- **Reads** (ro by Unix permissions, never by trusting the tool list):
  `/srv/vaults/freelance-practice` (exists) and `/srv/vaults/infra-docs` (new
  root mirror, own read-only deploy key).
- **Writes**: only `mcpread`'s own clone of `freelance-automation-inbox`
  (own write deploy key, hermes pattern), exposed as vault `inbox` →
  `inbox/chat/`; 15-min push timer. No vault push key on the public path
  (D4a: agents commit, humans push; laptop moves notes into vaults).
- **Ingress**: Tailscale Funnel → :8422 directly (tailscaled already runs; no
  new daemon or credential; panel firewall untouched). Fallback if claude.ai's
  OAuth flow misbehaves with Funnel: Cloudflare Tunnel at
  `mcp.neurogenesis.dev`.
- **Gate before exposure**: Codex review of `auth-server.js` (young project:
  v1.4.2, single author). Rebuild/update on upstream releases is the care task.
- **Kill switch**: `tailscale funnel --https=443 off` + `systemctl stop
  mcpread-auth mcpread-gateway` from the admin lane (unit names as built).

## Rejected

- **Vault push key for the endpoint** — an unattended, publicly reachable
  process holding a key that rewrites the vaults; prompt injection in a chat
  becomes silent record rewriting. The inbox *is* the branch-and-merge pattern
  for unattended principals.
- **Headless Obsidian Sync node on the VPS** — joins the paid Sync mesh and
  writes into laptop *and* phone in real time: an upward write channel
  (ADR-0027), plus Electron-under-xvfb as a service.
- **Endpoint on the phone** — Android kills background servers; a public
  surface on the most-trusted device inverts the trust order.
- **Cloudflare Tunnel as primary** — works, but adds a daemon and a CF
  credential the box doesn't otherwise need. Kept as fallback.

## Amendments while building (2026-09-20, stages 1–2)

- No Claude Code login as `mcpread`: install by paste, review as `dev` on its own clone.
- `obsidian-mcp` refuses read-only vaults at startup; patched locally to `R_OK`
  (`patch-ro-vaults.sh`), so "read-only by permissions" holds at call time, not at start.
- Review gate result: four hardening commits on `auth-server.js` (redirect-host allowlist,
  async scrypt + login/registration limiters, client-row cap with grace, `trust proxy`
  loopback). **Accepted residual risk:** an unauthenticated caller can hold login or
  registration at 429 for 15 minutes — availability only, single user, ntfy + kill switch.
- Funnel serves only 443/8443/10000 and 443 was `tailscale serve` (n8n) → first tried
  public port 8443. **Measured: claude.ai connectors never attempted a connection to a
  non-443 port** (tcpdump on the loopback upstream stayed silent during Reconnect) →
  the MCP took 443, n8n's tailnet-only `serve` moved to 8443 (n8n env + a second Google
  OAuth redirect URI). Public URL `https://netcup-vps.tail82bb86.ts.net/mcp`.
- Ingress verified: the Funnel name resolves to Tailscale ingress addresses
  (`185.40.234.x`), tailscaled listens on tailnet addresses only, the VPS's public IP
  exposes nothing new. The auth server sees the visitor address via `X-Forwarded-For`.
- Accepted 2026-09-20 after the acceptance table in `../runbooks/mcp-endpoint.md`
  passed in full (eight items, incl. the off-tailnet scan of the public IP: 443, 8443,
  22 all closed).

## Consequences and risks

- Public attack surface = auth-server.js and its OAuth endpoints. Bounded by:
  review before exposure, unprivileged principal, localhost-fenced upstream,
  egress drops, kill switch, update cadence.
- Prompt injection inside Luis's own chats can at worst vandalise the inbox
  repo (recoverable: git history + B2). It cannot touch vaults or credentials.
- Mirror staleness: answers are as fresh as the laptop's last push + mirror
  pull. Consider `vault-mirror.timer` hourly → 15 min.
- New ADR-0034 credential rows: `mcpread` inbox deploy key (write, inbox repo
  only); root ro deploy key for the infra-docs mirror; OAuth password hash +
  static token under `~mcpread/.config/obsidian-mcp/` (0600, excluded from
  backups as credentials).

## Build stages (each a stopping point)

1. This ADR + infra-docs mirror + `mcpread` principal + inbox clone (~45 min).
2. Install server as `mcpread`; Codex review; units with hc-run/ntfy pattern (~1 h).
3. Funnel + claude.ai connector; gate on the OAuth handshake (~45 min).
4. Acceptance (see backlog note) + ADR-0034 rows + runbook (~30 min).
