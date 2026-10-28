---
type: backlog-item
status: done  # 2026-09-20 — built per ADR-0037, runbook netcup-vps/runbooks/mcp-endpoint.md
priority: medium
device: netcup-vps
projects: [netcup-build]
tags: [backlog, mcp, claude-desktop, tailscale, funnel, containment]
created: 2026-09-15
updated: 2026-09-20
commit: pending
provenance: inferred
---

# mcp-read-access-from-desktop

> [!NOTE] Summary
> Luis wants Claude chats to read VPS files — on the laptop **and from the
> Claude phone app while the laptop is off** (requirement added 2026-09-15).
> That rules out anything the laptop launches. Only a **remote MCP connector**
> added in claude.ai works on every device, and claude.ai reaches connectors
> from Anthropic's cloud, so the server needs a **public HTTPS endpoint**.
> Proposed: read-only MCP server as principal `mcpread`, published with
> **Tailscale Funnel** (no panel-firewall change), **OAuth required**, scoped to
> the vault mirrors + inbox. ≈ 3–4 h. First public-facing service on this host —
> build it deliberately, not as a quick hack.

## Options considered

| Option | Phone, laptop off? | Public endpoint? | Verdict |
|---|---|---|---|
| Headless Claude Desktop on the VPS | no — chat UI is still on a client | no | rejected (GUI app; Linux build unofficial — verify) |
| Laptop Desktop → stdio MCP over SSH | **no** | no | rejected by the phone requirement; still the simplest laptop-only path |
| Claude app → Remote Control session (Claude Code as `dev`) | yes | no | works today; Code session, not a Project chat with memory/instructions |
| **Remote MCP connector via Tailscale Funnel + OAuth** | **yes** | yes (Funnel) | **proposed** |

## Design (draft)

- **Principal** `mcpread`: no SSH login, no sudo, `skuid` egress rules like `auto`,
  own slice with a small `MemoryMax`.
- **Scope, v1:** `/srv/vaults/*` (root-owned mirrors) and `/home/auto/inbox`
  (read via ACL). **Not** `~dev/dev` in v1 — code and client repos stay off the
  public path until the rest has run for a few weeks.
- **Read-only by permissions:** the server may expose write tools; Unix
  permissions make them fail. Prefer a server with a read-only mode (check).
- **Transport:** MCP server bound to `127.0.0.1:<port>` → `tailscale funnel`
  on 443 (or a dedicated funnel port). Funnel ingress arrives through
  Tailscale's relays, so the netcup `tailnet-only` policy is unchanged (verify
  with an off-tailnet `curl`). Loopback port fenced from `dev`/`auto`/`hermes`
  like n8n's (skuid drop).
- **Auth:** OAuth in front of the server (claude.ai custom connectors support
  OAuth — verify current support for client ID/secret in the connector
  settings). **No authless endpoint, ever**, even "temporarily".
- **Extra fences:** restrict source IPs to Anthropic's published outbound ranges
  if the proxy allows (verify the list exists and is current); rate limit;
  access log to journald; `mcp-alive.timer` → ntfy + healthchecks.
- **Kill switch:** `tailscale funnel reset` + `systemctl stop` the unit.
- Records: ADR (first public-facing service; amends the `tailnet-only`
  assumption of ADR-0029 at the tailnet layer, not the panel layer), row in
  ADR-0034 (OAuth client secret), listener inventory.

## Acceptance

From the phone app with the laptop off: read a vault-mirror note ✓. Unauthenticated
`curl` to the Funnel URL → 401 ✓. Tool call reading `~dev/.ssh/config` ✗,
`/etc/credstore/*` ✗; write under `/srv/vaults` ✗. `nc` from `mcpread` to the
laptop ✗ (counter increments). Off-tailnet port scan of the public IP unchanged.

## Spike results (2026-09-20)

- **claude.ai side confirmed** (help article, updated 2026-08): custom connector =
  remote MCP URL; OAuth client ID/secret optional under Advanced settings — DCR
  (the server registers Claude itself) is the default path. Connectors are
  account-level → work from the phone app. Known flakiness in Anthropic's OAuth
  orchestration with exotic IdP setups (github claude-ai-mcp issues #212, #313);
  the standard DCR + PKCE path per the MCP 2025-06-18 auth spec is the one that
  works → use a server that implements exactly that.
- **Base chosen: `rollecode/obsidian-remote-mcp`** — purpose-built for this:
  obsidian-mcp (stdio) → supergateway (Streamable HTTP :8420, localhost-only via
  unit IPAddressDeny) → auth-server.js (OAuth 2.1 + PKCE + DCR + refresh
  rotation, :8422). Reads vaults from disk (no Obsidian), serves several vaults
  from one endpoint, static bearer token alongside OAuth (Claude Code on the
  laptop could use the same endpoint). Node 18+ (VPS: 22). Systemd units + an
  optional nginx layer; "any HTTPS reverse proxy works".
  Caveats: young project (v1.4.2, single author, 0 stars) → **Codex review of
  auth-server.js before exposure**; tool set includes write/delete → read-only
  comes from Unix permissions (ro access for `mcpread`), never from the tool
  list.
- **Write surface** (decided with Luis 2026-09-20): vault mirrors stay ro —
  write tools fail on them by permission. `mcpread` gets its **own clone of
  freelance-automation-inbox** (own write deploy key, hermes pattern) exposed as
  vault "inbox"; a 15-min push timer mirrors hermes-inbox-push. Phone chats can
  save notes to `inbox/chat/`; Luis moves them into vaults deliberately.
  ADR-0027/D4a hold: no vault push key on the public path.
- **Vault scope v1**: `/srv/vaults/freelance-practice` (exists),
  `/srv/vaults/infra-docs` (new root mirror + ro deploy key, ~10 min), inbox
  clone (rw). Consider `vault-mirror.timer` hourly → 15 min for freshness.
- **Ingress**: Funnel and Cloudflare Tunnel both work (outbound-only, panel
  firewall unchanged). Lean **Tailscale Funnel** — tailscaled already runs, no
  new daemon, no CF credential on the VPS; Funnel can point at :8422 directly
  (nginx layer optional). CF Tunnel (`mcp.neurogenesis.dev`) is the fallback if
  Funnel + the OAuth redirects misbehave. Decide in the ADR.
- Cost €0/mo. Remaining unknown: none blocking — mirror lag stays the known
  caveat (show `updated` dates in answers).


## Related

- ADR-0027 (trust direction), ADR-0029 (sshd/nftables), ADR-0034 (credentials),
  ADR-0036 (secretary — similar containment pattern)
- Laptop side: the existing home-wide filesystem MCP should be narrowed (arch-thinkpad)
