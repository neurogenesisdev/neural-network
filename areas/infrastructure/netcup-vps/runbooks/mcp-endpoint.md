---
title: "MCP endpoint for claude.ai: mcpread, obsidian-remote-mcp v1.5.0, Funnel"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-10-29
commit: pending
created: 2026-09-20
updated: 2026-09-20
tags: [runbook, mcp, claude-ai, mcpread, funnel, containment, obsidian]
---

# MCP endpoint: `mcpread`, obsidian-remote-mcp, Funnel

> [!NOTE] Summary
> This runbook implements [[../decisions/20261029-claude-ai-vault-connector-read-write|the connector ADR]]
> (ADR-0041, supersedes [[../decisions/20260920-public-mcp-endpoint-claude-ai|ADR-0037]]).
> claude.ai reads and writes the live vault, and reads `/home/dev/code` read-only, through
> `https://netcup-vps.tail82bb86.ts.net/mcp`. Built 2026-10-28/29; acceptance passed
> 2026-10-29 from the phone with Tailscale off. Everything below is `measured` unless marked otherwise.
> **Kill switch:** `tailscale funnel --https=443 off && systemctl stop mcpread-auth mcpread-gateway`.

The ADR-0037 build (mirrors, inbox repo, v1.4.2, static token) is gone with the 2026-10
rebuild. Its runbook text is in the vault's git history before 2026-10-29.

Lanes: admin is `luis` with sudo. A shell as `mcpread` is `sudo -u mcpread -H bash`; it is a
`nologin` user, so `su` does not work.

## Layout

| Piece | Where | Notes |
|---|---|---|
| Principal | `mcpread`, uid 1003, groups `mcpread vault` | `nologin`, linger, no sudo, no Claude Code login. Slices `user-1003.slice` and `mcpread.slice`: `MemoryHigh=768M`, `MemoryMax=1G` |
| Server code | `~mcpread/obsidian-remote-mcp`, branch `local/netcup` | Upstream `v1.5.0` (`868e32e`) plus 11 local commits (below). `origin` is upstream, read-only. Node is Ubuntu's `nodejs` 22 |
| Gateway | `mcpread-gateway.service` | supergateway on **:8420, all interfaces** (no `--host` option), Streamable HTTP, `--stateful --sessionTimeout 3600000`, child `vault-mcp.js /srv/vault /srv/vaults/code` |
| Auth | `mcpread-auth.service` | `127.0.0.1:8422`, `UPSTREAM=http://127.0.0.1:8420`, `ISSUER=https://netcup-vps.tail82bb86.ts.net`, `DEFAULT_VAULT=vault`, `CONFIG_DIR=~mcpread/.config/obsidian-mcp` |
| Policy | `/etc/mcpread/vault-policy.json` (root 0644) | Loaded through `VAULT_POLICY`. The server will not start if the policy is unreadable, names an unserved vault, disables an unknown tool, or has no entry for a served vault |
| Code tree | `srv-vaults-code.mount` | `/home/dev/code` → `/srv/vaults/code`, `bind,ro,nosuid,nodev`. ACL `u:mcpread:rX`, access and default, on `/home/dev/code` |
| Password | `/root/mcpread/login-password` (root 0600) | KeePassXC **"netcup vaults MCP login"**. Hash: `~mcpread/.config/obsidian-mcp/password-hash` (0600, `salt:scrypt-64`). No static token |
| Alive check | `mcpread-alive.timer`, hourly ±5 min | `/usr/local/lib/mcpread/alive.sh` as `mcpread`; `OnFailure=ntfy-alert@%n.service` |
| Tailscale | 1.104.1, `apt-mark hold` | Held since 2026-10-29, so a Funnel change cannot alter the exposure unseen |

Both services run `User=mcpread`, `Slice=mcpread.slice`, `NODE_ENV=production`,
`OnFailure=ntfy-alert@%n.service`, `IPAddressDeny=any`, `IPAddressAllow=localhost`,
`ProtectSystem=strict`, `ProtectHome=tmpfs`, `CapabilityBoundingSet=` and the
`Protect*`/`Restrict*` family.

## Two fences

What the connector cannot do is enforced twice wherever the kernel can express it.

| Path | Kernel (gateway unit) | App policy |
|---|---|---|
| `.claude`, `_generated`, `_meta/scratch`, `_meta/legacy-rules` | `InaccessiblePaths` | deny |
| `.obsidian` | none | deny; upstream also refuses writes there |
| `.git`, `.env*` (both vaults) | none; the vault has no `.git` | deny |
| `areas/infrastructure/arch-thinkpad/state` | `ReadOnlyPaths` | read-only (`areas/infrastructure/*/state`) |
| `CLAUDE.md`, `AGENTS.md`, every `_index.md` | none | read-only |
| code vault | the `ro` bind mount | read-only `**`; deny `node_modules`, `dist` |
| `/var/lib/vault-git`, `/etc/credstore`, `/root` | `InaccessiblePaths` | none (outside the vaults) |
| `move-directory`, `delete-directory` | none | disabled. `delete-note` and `move-note` refuse folders and links to folders |

Links are followed only to allowed targets inside the vault, and never written through.
`delete-note` removes a link itself; links are never sent to `.trash`. Tool errors that
are not the server's own messages reach the caller as `The server could not complete this
(<code>).`, so host paths never leave the server. A denied path that the kernel blocks first
shows up as `EACCES`. That is expected: the kernel answers before the policy does.

**Network.** The `/etc/nftables.conf` output chain has two kinds of rule.
- Before the established and loopback accepts: `oif lo tcp dport { 8420, 8422 } meta skuid
  != { root, mcpread } drop`.
- Drops for `skuid mcpread` to RFC 1918, 169.254/16 and 100.64/10, and for IPv6 to
  `fc00::/7` and `fe80::/10`.

The input chain has `tcp dport { 8420, 8422 } drop` after `iif lo accept`. That rule is
what closes supergateway's wildcard bind to the tailnet. Backup of the pre-change file:
`/root/nftables.conf.bak-2026-10-28`.

## Funnel

```bash
# root. Six path mounts on 443, nothing else; n8n's tailnet-only serve on 8443 stays.
for p in /mcp /.well-known/oauth-protected-resource /.well-known/oauth-authorization-server /register /authorize /token; do
  tailscale funnel --bg --yes --https=443 --set-path="$p" "http://127.0.0.1:8422$p"
done
tailscale funnel status
```

Each mount also covers everything below its path, so the two `.well-known` mounts serve
their `/mcp` variants. The login page's `/icon.svg` is not mounted, so the page shows no
logo. That is cosmetic.

Gotcha: for about a minute after the first `funnel` command, public requests fail during
the TLS handshake (curl exit 35, `000`) while Tailscale issues the certificate.

Gotcha: on this host the Funnel name resolves to the tailnet address, so a plain `curl`
from here tests `serve`, not Funnel. To take the real public route, ask public DNS (`dig` is
not installed) and pin the answer:

```bash
H=netcup-vps.tail82bb86.ts.net
PUB=$(curl -s -H 'accept: application/dns-json' "https://cloudflare-dns.com/dns-query?name=$H&type=A" | jq -r '[.Answer[]?|select(.type==1)|.data][0]')
curl -s --resolve "$H:443:$PUB" -o /dev/null -w '%{http_code}\n' -X POST "https://$H/mcp"    # 401
```

## Connector

claude.ai → Settings → Connectors → Add custom connector. Name `netcup vaults`, URL
`https://netcup-vps.tail82bb86.ts.net/mcp`. Leave OAuth client ID and secret empty, since
Claude registers itself through DCR. The sign-in page asks for the password in KeePassXC
"netcup vaults MCP login". Connectors belong to the account, so the phone app has the
connector as soon as it is added on the web.

Reading the password on the server: `sudo cat /root/mcpread/login-password`. **In zsh, a
highlighted `%` appears after it. That `%` is zsh's no-trailing-newline marker, not part of
the password.**

## Local commits on `local/netcup`

| Commit | What |
|---|---|
| `8ada6f6` `173e805` `63dde07` `4ac37dc` | ADR-0037's auth-server hardening, carried to v1.5.0: redirect-host allowlist, login and registration limiters, client-row cap with grace, `trust proxy` loopback. The only copy was restic snapshot `906868b7`, restored from it on 2026-10-28 |
| `b45987b` | The access policy: deny, read-only, disabled tools |
| `a47b088` | Locked dependencies updated to clear audit advisories. Residual `qs` and `zod` in supergateway were reviewed and found unreachable |
| `9cfec58` | Links followed only to allowed targets; refuse a policy naming an unserved vault |
| `faaa1b5` | A successful sign-in clears only its own address window, not the global count |
| `6268ecf` | Act on links themselves, refuse folders in note tools, check the `.trash` path, per-path `read-notes`, require a policy entry for every served vault |
| `76beaf7` | Fixed text for host errors, refuse links to folders, limiter argument checks, `test/auth.js` |
| `830becb` | Report invalid frontmatter YAML to the caller |

Four adversarial reviews: the full `v1.5.0..local/netcup` diff plus upstream's auth-server
change, then commits 7–8, 9 and 10 separately. None found anything blocking. The open
Consider items, none urgent, are:
- `test/auth.js` inherits the shell's environment and discards server output.
- The registration test does not prove the limit is per address.
- The folder-link refusal lets the unlink through when `stat` fails with `EACCES`.

**These commits exist only in this clone and in restic.** Moving them into infra-tools
(patches plus an apply script) is a laptop item on [[vault-merge-2026-10]].

Tests: `npm test` runs `test/run.js` (84), `test/policy.js` (34), `test/limiter.js` (6) and
`test/auth.js` (2).

## Acceptance (2026-10-29)

| # | Check | Result |
|---|---|---|
| 1 | Laptop over Tailscale → `:8420`, `:8422` | `000` after 5 s each; input drop counter 0 → 15 |
| 2 | `dev`, `luis`, `n8n` → `127.0.0.1:8420` | blocked by the loopback fence |
| 3 | Public route, `/mcp` without a token | 401 with `WWW-Authenticate: Bearer resource_metadata=…` |
| 4 | Public metadata | issuer and resource correct, `code_challenge_methods_supported: ["S256"]` |
| 5 | Public route, unmounted paths (`/`, `/healthz`, `/icon.svg`, `/.git/config`, `/oauth.db`, `/mcp/../healthz` …) | all 404; `:8443` does not answer from outside; `:80` 302 (Funnel's https redirect) |
| 6 | Forged `X-Forwarded-For` on a wrong-password sign-in | logged under the real sender's address, not the forged one; the throwaway client was deleted afterwards |
| 7 | Live gateway tool checks | 34 tools, directory tools absent; denied and read-only paths refused; code vault refuses deletes; batch read keeps going past bad paths |
| 8 | claude.ai, phone on mobile data with Tailscale off (Luis) | stats and reads work; `_generated/.keep` → EACCES with no path; `CLAUDE.md` append refused; code-vault write refused; code listing hides `.git`, `.env`, `node_modules` |
| 9 | Sign-in source in the auth log | an outside address, not loopback |
| 10 | `mcpread-alive` | exit 0; a copy pointed at an unused port exits 1 with `FAIL: gateway /healthz` |

## Checks and gotchas

- `curl` to 8420/8422 **as `luis` hangs**: the loopback fence drops it. Check as root or
  `mcpread`, with `-m5`: `sudo -u mcpread curl -m5 -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8420/healthz`.
- `vault-mcp` children belong to gateway sessions and are reaped after the 1 h session timeout.
- The log line on each new session reads `serving 2 vault(s): vault, code - 34 tools - policy on`
  (`journalctl -u mcpread-gateway`). Without `policy on`, stop the endpoint.
- A wrong password shows as `login failed from <address>` in `journalctl -u mcpread-auth`.
  A successful sign-in is not logged.
- OAuth state is `~mcpread/.config/obsidian-mcp/oauth.db` (better-sqlite3; no `sqlite3` CLI on
  the host). To revoke every client and token, stop `mcpread-auth`, move the file away, and
  start the service again. The connector then has to sign in again.

## Care

**Monthly: upstream rebase.**
1. As `mcpread`, run `git -C ~/obsidian-remote-mcp fetch --tags origin` and look for a release after `v1.5.0`.
2. Rebase `local/netcup` on the new tag in a scratch clone and run `npm ci && npm test`.
3. Review the diff adversarially if `auth-server.js`, `lib/paths.js`, `lib/fsops.js` or a
   write tool changed.
4. Bring the commits across with `git format-patch` → `git am` as `mcpread`, then restart
   both units and re-run acceptance items 3–5 and 7.
5. If the new release adds a pre-write hook, revisit the ADR: run `vault-snap` before each write.

**On a Tailscale update** (held): unhold, upgrade, run `tailscale funnel status`, then
acceptance items 3–5. Hold again.

**Kill switch:** `tailscale funnel --https=443 off && systemctl stop mcpread-auth mcpread-gateway`.
n8n's serve on 8443 is unaffected. Re-enable with `systemctl start mcpread-gateway
mcpread-auth`, then the Funnel loop above.

## Related

- [[../decisions/20261029-claude-ai-vault-connector-read-write|Connector ADR]] (ADR-0041) ·
  [[../decisions/20260920-public-mcp-endpoint-claude-ai|ADR-0037]] (superseded on acceptance)
- [[adr-0040-vault-merge-and-agent-council|ADR-0040]]: vault git and `vault-snap`, the recoverability this relies on
- [[restic-b2-backup]]: where the local commits are backed up until infra-tools has them
