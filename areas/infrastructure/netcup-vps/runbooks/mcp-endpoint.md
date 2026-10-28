---
title: "MCP endpoint for claude.ai: the mcpread principal, mirrors, server, Funnel"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-20
commit: pending
created: 2026-09-20
updated: 2026-09-20
tags: [runbook, mcp, claude-ai, mcpread, funnel, containment, obsidian]
---

# MCP endpoint — `mcpread`, vault mirrors, obsidian-remote-mcp, Funnel

> [!NOTE] Summary
> Implements [[../decisions/20260920-public-mcp-endpoint-claude-ai|ADR-0037]] in four
> stages. **All four stages done 2026-09-20, every command run and its result seen;
> acceptance passed from the phone with the laptop offline.** Public URL:
> `https://netcup-vps.tail82bb86.ts.net/mcp` (Funnel :443). All eight acceptance items
> measured. Design and spike: `../backlog/mcp-read-access-from-desktop.md`.

Lanes as in [[automations-stage-2|stage 2]]: **admin** = `luiswg` + `sudo -i`. A
shell as `mcpread` is `sudo -u mcpread -H bash` from the admin lane (nologin shell,
so `su - mcpread` does not work; `-H` sets `$HOME`). One shell-starting line per
paste block.

## What mcpread can reach (stage 1 state)

| Path | Mechanism | Write? |
|---|---|---|
| `/srv/vaults/freelance-practice` | root-owned mirror, world-readable; root's ro key pulls it hourly | no (measured: Permission denied) |
| `/srv/vaults/infra-docs` | same, new 2026-09-20 | no (measured) |
| `/home/mcpread/inbox` | own clone, own **write** deploy key | yes — `chat/` is the target for notes saved from chats |
| `~dev/.ssh/config`, `/etc/credstore/` | — | denied (measured) |
| tailnet / RFC 1918 / CGNAT | nftables `skuid` drop | no (measured: `nc` to the laptop times out, counter +3) |

## Stage 1 — mirror + principal + inbox clone (done 2026-09-20)

### B — `/srv/vaults/infra-docs` root mirror

Admin lane (root): `ssh-keygen -t ed25519 … -f /root/.ssh/gh-infra-docs-mirror`,
`Host gh-infra-docs-mirror` block appended to `/root/.ssh/config`; deploy key added
**read-only** on `neurogenesisdev/infra-docs` via the GitHub web UI (the form worked
this time; `gh repo deploy-key add` from the laptop remains the fallback). Then:

```bash
ssh -T gh-infra-docs-mirror; echo "exit $?"                                                    # Hi neurogenesisdev/infra-docs! exit 1
git ls-remote gh-infra-docs-mirror:neurogenesisdev/freelance-automation-inbox.git; echo "exit $?"   # must FAIL: 128
git clone -q gh-infra-docs-mirror:neurogenesisdev/infra-docs.git /srv/vaults/infra-docs
M=$(stat -c '%a' /srv/vaults/freelance-practice); chmod "$M" /srv/vaults/infra-docs; chown -R root:root /srv/vaults/infra-docs
systemctl start vault-mirror.service; echo "exit $?"      # vault-mirror.sh globs /srv/vaults/*/ — nothing to edit
su - auto -c 'head -3 /srv/vaults/infra-docs/netcup-vps/_index.md; touch /srv/vaults/infra-docs/x'   # readable; Permission denied
```

Measured: both mirrors `drwxr-xr-x root root`; `vault-mirror.service` exit 0, no
`pull failed`; mirror HEAD `b3f5dd0` (= the morning's ADR commit); `auto` reads, cannot write.
Freshness = laptop `vault-push.timer` (hourly) + `vault-mirror.timer` (hourly); shorten
the latter to 15 min if answers feel stale.

### C — principal `mcpread`

Admin lane (root):

```bash
useradd -m -s /usr/sbin/nologin -c 'MCP endpoint principal (claude.ai)' mcpread     # uid 1005
loginctl enable-linger mcpread
install -d -m 755 /etc/systemd/system/user-1005.slice.d
printf '[Slice]\nMemoryHigh=768M\nMemoryMax=1G\n' > /etc/systemd/system/user-1005.slice.d/50-mem.conf
printf '[Unit]\nDescription=Resource envelope for mcpread system units\n\n[Slice]\nMemoryHigh=768M\nMemoryMax=1G\n' > /etc/systemd/system/mcpread.slice
systemctl daemon-reload
install -d -m 700 -o mcpread -g mcpread /home/mcpread/.ssh /home/mcpread/.config
cp -a /etc/nftables.conf /root/nftables.conf.bak-$(date +%F)
sed -i -E '/skuid \{/{/"mcpread"/!s/"hermes"/"hermes", "mcpread"/}' /etc/nftables.conf
nft -c -f /etc/nftables.conf && systemctl reload nftables
```

Two slices on purpose: `user-1005.slice` caps anything under the login session
(the stage-2 Claude Code session), `mcpread.slice` is for system units with
`User=mcpread` (auto pattern). Same limits, so it does not matter where a process lands.
`"mcpread"` is in all three `skuid` sets — including the n8n loopback fence (port 5678):
the MCP principal has no business talking to n8n. **The sets live in
`/etc/nftables.conf`**, not in an `nftables.d/` fragment (a wrong path cost one round —
the reload was a silent no-op and `mcpread` was briefly unfenced).

As `mcpread` (`sudo -u mcpread -H bash`): keygen `~/.ssh/gh-automation-inbox`
(comment `netcup-vps mcpread - write - freelance-automation-inbox`), alias
`Host gh-automation-inbox`, `ssh-keyscan` known_hosts (fingerprint checked against
GitHub's published Ed25519 key), git identity `mcpread (netcup-vps)
<mcpread@netcup-vps.invalid>`, `pull.ff only`. Deploy key added on
`neurogenesisdev/freelance-automation-inbox` with **Allow write access**. Then:

```bash
ssh -T gh-automation-inbox; echo "exit $?"                                                # Hi …/freelance-automation-inbox! exit 1
git ls-remote gh-automation-inbox:neurogenesisdev/freelance-practice.git; echo "exit $?"  # must FAIL: 128 — one key, one repo
git clone -q gh-automation-inbox:neurogenesisdev/freelance-automation-inbox.git ~/inbox
cd ~/inbox && mkdir -p chat && touch chat/.gitkeep && git add chat/.gitkeep && git commit -qm 'chat: create inbox/chat (mcpread)' && git push -q
nc -zvw3 github.com 22                                                                    # succeeded
nc -zvw3 <TS_IPV4_arch-thinkpad> 22                                                       # laptop tailnet IP (tailscale status): timed out
```

Measured 2026-09-20: clone on `master`; `e16e5f3 chat: create inbox/chat (mcpread)`
pushed; github.com:22 reachable; laptop `nc` timed out and the `ip daddr … counter drop`
rule went `packets 0` → `packets 3`; vault mirrors readable, `touch` denied;
`~dev/.ssh/config` and `/etc/credstore/` denied; `Linger=yes`; both slices
`MemoryMax=1073741824`. Public keys + fingerprints in KeePassXC; private keys only on the host.

Gotcha for the negative test: a placeholder like `<TS_IPV4>` in a paste is a shell
redirect — the command fails with "No such file" and the counter stays 0, which looks
like a pass if you only read `exit 1`. Substitute the real IP at paste time
(`tailscale status | grep thinkpad`) — the literal never goes into this repo (pre-commit hook).

## Stage 2 — server as `mcpread` (done 2026-09-20)

**Deviation from the ADR:** no Claude Code login as `mcpread` — the install is `git clone` +
`npm install` by paste, and the code review ran as `dev` (who already has Codex) on
`dev`'s own clone `~dev/dev/review/obsidian-remote-mcp`. One credential less on the
public principal.

### 2A — install (as `mcpread`)

`~/obsidian-remote-mcp` = upstream tag **v1.4.2 = `fbc35d4`**, `npm install` (147 packages,
0 vulnerabilities, Node 22.23.2 system-wide), `node set-password.js` (login password in
KeePassXC `obsidian-mcp login`), `openssl rand -hex 32 > ~/.config/obsidian-mcp/token`
(static bearer for laptop Claude Code; attribute of the same entry). Dir 0700, files 0600.

Two things found by running the stdio server against the mirrors:

1. **obsidian-mcp demands `W_OK` on every vault at startup** (`build/main.js:15666`,
   `fs.access(path, R_OK | W_OK)`), which contradicts the ro-mirror design. Patched to
   `R_OK`; `~/obsidian-remote-mcp/patch-ro-vaults.sh` re-applies it after any
   `npm install` (the patch lives in `node_modules`). Effect: write tools fail at call time
   on the mirrors with EACCES instead of the server refusing to start — the behaviour the
   ADR wants; the inbox clone stays writable.
2. **Every vault needs `.obsidian/app.json`** — the mirrors carry it, the inbox clone did
   not: `{}` stub committed to the inbox repo (`a4e9434`).

Measured: `Vault "freelance-practice"|"infra-docs"|"inbox" registered`, 11 tools
(create/edit/delete/move-note, search-vault, read-note, tags, create-directory,
list-available-vaults), clean SIGTERM.

### 2B — review gate (as `dev`, Codex `adversarial-review --base fbc35d4`)

Four rounds, four commits on branch `fix/exposure-hardening` (`auth-server.js` only),
moved to `mcpread`'s clone with `git format-patch` → `git am` (root copies the patch
files between homes). What changed and why:

| Finding (Codex) | Fix |
|---|---|
| Open DCR: anyone registers a client with his own callback, sends the owner a login link, gets a code | `ALLOWED_REDIRECT_HOSTS` (default `claude.ai,claude.com`), https only, ≤5 URIs, checked at `/register` **and** `/authorize`; login page shows client name + callback host |
| Unthrottled sync `scryptSync` on the event loop | async `crypto.scrypt`; login limiter per address 10 / 15 min + global `MAX_LOGIN_ATTEMPTS_GLOBAL`=100 |
| Unbounded client rows | `MAX_CLIENTS`=100, evict the oldest unused client older than `REGISTRATION_GRACE_SECONDS`=600 (503 if none); `purgeExpired` drops unused clients after 24 h |
| Eviction could delete a client mid-login | the grace period above + registration limiter per address 10 / global `MAX_REGISTRATIONS_GLOBAL`=100 per 15 min |
| All visitors share one address bucket behind a loopback proxy | `app.set('trust proxy', TRUST_PROXY‖'loopback')` — tailscaled is the only proxy; `req.ip` = rightmost untrusted `X-Forwarded-For` |

Verified by inspection (Codex, round 1): PKCE S256 enforced, `timingSafeEqual`, codes
hashed + single-use + ~60 s, exact `redirect_uri` match, refresh rotation, tokens and
client secrets hashed in SQLite, no eval/exec/child_process, no request-controlled file
read; outbound calls only to `UPSTREAM`.

**Gate rule adopted after round 4:** ship when no finding grants access without the
password or bypasses PKCE/redirect/token checks. Availability findings (an
unauthenticated caller can hold login or registration at 429 for 15 min) are accepted for
a single-user endpoint with ntfy on unit failure and a kill switch. Codex's last High
concerned upstream's nginx config, which is not deployed here.

Care: on an upstream release, `git fetch && git rebase v1.x.y` in both clones, re-run the
review if `auth-server.js` changed, `npm install`, `./patch-ro-vaults.sh`, restart both
units. Consider offering the four commits upstream as a PR.

Gotcha: Codex's backend is `chatgpt.com`; Claude Code's sandbox blocked it and the review
hung 600 s. Fixed by `sandbox.network.allowedDomains` in `~dev/.claude/settings.json`
(`chatgpt.com`, `*.chatgpt.com`, `api.openai.com`, `auth.openai.com`, `registry.npmjs.org`).

### 2C — units + fence (root)

System units `mcpread-gateway.service` (supergateway :8420, `IPAddressDeny=any` +
`IPAddressAllow=localhost`, `ReadWritePaths=/home/mcpread/inbox`) and
`mcpread-auth.service` (`Requires=mcpread-gateway`, `PORT=8422 UPSTREAM=http://127.0.0.1:8420
ISSUER=https://netcup-vps.tail82bb86.ts.net:8443 DEFAULT_VAULT=freelance-practice`,
`ReadWritePaths=/home/mcpread/.config/obsidian-mcp`); both `User=mcpread
Slice=mcpread.slice OnFailure=ntfy-alert@%n.service NoNewPrivileges PrivateTmp
ProtectSystem=strict ProtectHome=read-only`. `ISSUER` is `:8443` because Funnel serves only
443/8443/10000 and 443 is `tailscale serve` (n8n). Loopback fence added to
`/etc/nftables.conf` line 27: `meta skuid { "dev", "auto", "hermes", "n8n" } oif lo tcp
dport { 8420, 8422 } counter drop`.

Measured 2026-09-20: both `active`; `:8422` bound to `127.0.0.1` by the auth server
itself, `:8420` on `*` (supergateway has no `--host`) but unreachable except from loopback;
`/healthz` 200; `/mcp` without token 401; discovery document at
`/.well-known/oauth-authorization-server` with the `:8443` issuer; `dev` → `:8420` exit 28
(timeout); `initialize` with the static token → 200 through auth → gateway → obsidian-mcp.

### 2D — inbox push (root)

`/usr/local/lib/mcpread/inbox-push.sh` (pull → `git add chat` → commit → push),
`mcpread-inbox-push.{service,timer}` (15 min, `hc-run.sh`, `LoadCredential=hc-ping` ←
`/etc/credstore/hc-mcpread-inbox-push`, check `mcpread-inbox-push` 15 min / 15 min).
Measured: first run exit 0 in 3 s, commit `4b30757 chat: 2026-09-20T17:55:25Z`
(`chat/push-test.md` is the canary), timer next +15 min.

Kill switch (stage 2 state): `systemctl stop mcpread-auth mcpread-gateway`.

## Stage 3 — ingress (done 2026-09-20)

**Measured fact that changed the plan: claude.ai connectors only reach port 443.** With
Funnel on `:8443`, every endpoint was spec-correct off-tailnet (401, `WWW-Authenticate`
→ protected-resource metadata → auth-server metadata → `/register`), yet claude.ai
reported "Couldn't connect to the server" and `tcpdump -i lo port 8422` saw **no
connection attempt at all** during Reconnect. Moved to 443; the connector form then
showed *Detected* for OAuth and DCR at once. Funnel is per port, so n8n's tailnet-only
`serve` moved from 443 to **8443**.

```bash
# root — final layout
tailscale funnel --https=8443 off; tailscale serve --https=443 off
tailscale serve  --bg --https=8443 http://127.0.0.1:5678     # n8n, tailnet only
tailscale funnel --bg --https=443  http://127.0.0.1:8422     # MCP auth server, public
tailscale funnel status
```

n8n side: `~n8n/n8n.env` `N8N_HOST`, `N8N_EDITOR_BASE_URL`, `WEBHOOK_URL` → `…ts.net:8443`,
`systemctl restart user@1004.service`; Google OAuth client `n8n-neurogenesis` got a second
redirect URI `https://netcup-vps.tail82bb86.ts.net:8443/rest/oauth2-credential/callback`
(old one kept; the existing Gmail grant keeps working). `mcpread-auth.service`
`ISSUER=https://netcup-vps.tail82bb86.ts.net` (no port), restart.

DNS: the public A records for the Funnel name (`185.40.234.x`, Tailscale ingress — not
the VPS) appeared ~5 min after the first `funnel` command; `ts.net` answered NXDOMAIN
until then, and 1.1.1.1 caches that negative answer for 300 s — query `dns.google` to
retry. `ss` shows tailscaled listening on the **tailnet** addresses only; internet traffic
arrives through Tailscale's relays, the panel firewall is unchanged.

Connector (claude.ai web, Settings → Connectors → Add custom connector): name
`netcup vaults`, URL `https://netcup-vps.tail82bb86.ts.net/mcp`, Authentication *Sign in
now*, OAuth client *Register automatically (DCR)*, no headers, transport *Streamable HTTP*
— all three "Detected". Connect → the server's login page (shows client "Claude" and
callback host `claude.ai`) → password from KeePassXC `obsidian-mcp login`. Connectors are
account-level: the phone app has it immediately. The phone app's connector UI is limited
(no delete) — manage connectors on the web.

Measured: `login failed from <TS_IPV4_arch-thinkpad>` for a deliberate wrong password from the laptop
(on-tailnet, so the tailnet address) — the auth server sees the visitor address, not
`127.0.0.1`; per-address limits are real. Nothing on the VPS's own public IP changed.

## Stage 4 — acceptance (done 2026-09-20)

| # | Check | Result |
|---|---|---|
| 1 | Phone app, laptop fully offline: read a vault note | ✓ `list-available-vaults` → three; `read-note _index.md` from `freelance-practice` and `infra-docs` |
| 2 | Unauthenticated `curl` to the public URL → 401 | ✓ off-tailnet, valid Tailscale cert |
| 3 | Path traversal (`../../home/dev/.ssh/config`) | ✓ refused: "not found in vault" (server sandboxes to the vault root; EACCES behind it) |
| 4a | Write under `/srv/vaults` | ✓ refused: `EROFS: read-only file system` (`ProtectSystem=strict` fences before Unix perms) |
| 4b | Write `inbox/chat/…` | ✓ `chat/2026-09-20-acceptance-test.md` created |
| 5 | `nc` from `mcpread` to the laptop | ✓ timed out, drop counter +3 (stage 1) |
| 6 | Off-tailnet scan of the VPS public IP: 443, 8443, 22 closed | ✓ all three time out (phone, mobile data, Tailscale off); a first attempt with Tailscale on "connected" — that was the tailnet listener, not the public IP |
| 7 | Inbox note reaches GitHub | ✓ `7fc41fd chat: 2026-09-20T18:59:35Z` |
| 8 | Visitor address in the auth log | ✓ |

Laptop Claude Code (static token, same endpoint):

```bash
claude mcp add --transport http netcup-vaults https://netcup-vps.tail82bb86.ts.net/mcp --header "Authorization: Bearer <token from KeePassXC>"
```

**Kill switch:** `tailscale funnel --https=443 off && systemctl stop mcpread-auth mcpread-gateway`
(n8n's `serve` on 8443 is unaffected). Re-enable: `systemctl start mcpread-gateway mcpread-auth &&
tailscale funnel --bg --https=443 http://127.0.0.1:8422`.

### Freshness and monitoring (done 2026-09-20, 22:00)

Bus scenario: end a laptop session, continue on the phone 15 min later with current
vault content. The chain is laptop `vault-push.timer` → GitHub → `vault-mirror.timer`,
so **both** run every 15 min now (worst case ~30 min lag, was ~2 h):

- laptop: drop-in `~/.config/systemd/user/vault-push.timer.d/15min.conf`
  (`OnUnitActiveSec=15min`); after `daemon-reload` a timer with only `OnUnitActiveSec`
  shows `NEXT -` until `systemctl --user restart vault-push.timer` (measured).
- VPS: drop-in `/etc/systemd/system/vault-mirror.timer.d/15min.conf`
  (`OnCalendar=*:02/15`, `RandomizedDelaySec=1min`).

`mcpread-alive.timer` hourly → `/usr/local/lib/mcpread/alive.sh` as `mcpread` via
`hc-run.sh` (check `mcpread-alive`, 1 h / 30 min, `/etc/credstore/hc-mcpread-alive`):
gateway `/healthz` 200, auth `/mcp` 401 locally **and via the public URL** (the VPS resolves
the Funnel name to Tailscale's ingress, so this exercises the real internet path), prints
the mirror HEAD dates. Gotcha: git refuses root-owned repos for another user ("dubious
ownership") → `git -c safe.directory=<path>` in the script. Measured: exit 0; with
`mcpread-auth` stopped exit 1 → ntfy + healthchecks on the phone.

Canaries `chat/push-test.md`, `chat/2026-09-20-acceptance-test.md` removed from the inbox
repo (`462d5ce`, laptop clone).

**Care:** monthly — check upstream releases (`git -C ~mcpread/obsidian-remote-mcp fetch --tags`),
rebase the four hardening commits, `npm install && ./patch-ro-vaults.sh`, restart, re-run
acceptance #2/#4. Backlog: offer the hardening commits upstream.

## Related

- [[../decisions/20260920-public-mcp-endpoint-claude-ai|ADR-0037]] — the decision
- [[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — credential rows (two added for stage 1)
- [[automations-stage-2|automations stage 2]] — `auto`/`hermes` patterns this copies
