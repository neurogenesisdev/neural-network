---
type: design
status: in-progress  # 2.4 waiting, 2.5 parked
provenance: inferred
device: netcup-vps
created: 2026-09-12
updated: 2026-09-13
tags: [automation, agents, systemd, ntfy, hermes, n8n, containment]
---

# Stage 2 — automations: producers, an inbox, and a secretary

> [!NOTE] Summary
> Stage 2 of `2026-09-10-ai-coding-rig-and-automation-plan.md`, reshaped on
> 2026-09-12 around one requirement: Luis wants a chat-first assistant he can
> talk to about work and administration, not only scheduled jobs. Design:
> **producers** (systemd timers as `auto`: Python for deterministic work,
> `claude -p` for judgment) write dated notes into one **inbox repo**; a
> **secretary** (Hermes Agent, contained as a fourth principal, no skills,
> tailnet-only gateway) reads that inbox and the vault clones and answers on
> Telegram. Each piece has one job; the always-on piece holds the fewest
> credentials. Everything below is `inferred` except where marked done. Ordered so
> each step is useful on its own and teaches something sellable.

## 0. What already exists (measured, stage 0/1, 2.0)

`auto` user (no SSH login, uid 1002, egress-restricted); ntfy topic on the phone
with `ntfy-alert@.service` (topic-only form, `LoadCredential=ntfy-topic`); restic daily; `dev`'s read-only vault clones; the
`skuid` egress rules; podman rootless; Node 22, Python 3, Chromium.

**Done 2026-09-12 (2.0):** `auto` bootstrapped — write deploy key to
`Luis-Gordon/freelance-automation-inbox` (measured refused on the vault repo),
clone at `/home/auto/inbox`, root commit `818bf09` on `master`, linger on,
`auto.slice` (MemoryHigh 1.5G / Max 2G). **Jobs run as system units with
`User=auto`** (`/etc/systemd/system/auto-*.service`), not user units: pattern
`Slice=auto.slice OnFailure=ntfy-alert@%n.service NoNewPrivileges PrivateTmp
ProtectSystem=strict ReadWritePaths=/home/auto/inbox`, scripts root-owned in
`/usr/local/lib/auto/`, API keys via `LoadCredential=` from `/etc/credstore/`.
Canary `auto-inbox-selftest.service` proven both ways (push succeeded; with the
key removed the failure reached the phone). Runbook:
`infra-docs/netcup-vps/runbooks/automations-stage-2.md`; sources in
`infra-docs/netcup-vps/build/auto/`.

## 1. Principals and data flow

```
timers (auto) ──write──▶ freelance-automation-inbox (git, own deploy key, write)
                                 │ pull                          ▲ pull, read-only
laptop Obsidian ◀── you move notes into the vault ──┘            │
                                                                 │
Telegram ◀──▶ hermes (container, principal `hermes`) ── reads inbox + vault clones (ro)
                     │ own API keys, spend-capped; no host home; no skills
                     └─ writes ONLY to inbox repo `secretary/` (own key)
n8n (later, principal `n8n`, own podman network) ── webhooks/files, no code or vault mounts
```

Rules: `auto` and `hermes` hold **no key to any vault**; the inbox repo is the
only thing they can write, and you move material into the vault deliberately.
A prompt injection through a scraped page or a Telegram message can at worst
vandalise the inbox. Every automation: trigger, inputs, outputs, failure
behaviour, alert, runbook, credential location (ADR-0034 table), cost/month.

## 2. The automations, in build order

| # | Automation | Trigger → output | Tool / model | Failure → alert | h | €/mo |
|---|---|---|---|---|---|---|
| 2.0 | **`auto` bootstrap + inbox repo** — **done 2026-09-12** (system units with `User=auto`, see §0) | — | `~auto/.ssh/gh-automation-inbox` (write), clone, `git config`, linger, `auto.slice`, unit pattern with the root `ntfy-alert@%n` template; laptop clones it too | canary `auto-inbox-selftest.service` | 1 | 0 |
| 2.1 | **Heartbeat** — **done 2026-09-13** | every timer run → `healthchecks.io` ping (free); one check per unit, URL in `/etc/credstore/hc-<unit>` via `LoadCredential=` | `hc-run.sh` wrapper (`/start`, run, `/<rc>` with output as body) — **not** `ExecStopPost=`: it has no `$CREDENTIALS_DIRECTORY` (measured) | missed ping → healthchecks alert — the watcher's watcher | 0.5 | 0 |
| 2.2 | **Trend digest** — **done 2026-09-13** | daily 06:00 → `inbox/digest/YYYY-MM-DD.md`; ntfy push with top titles + `Click` to the GitHub-rendered file (phone: tap → GitHub app, measured) | `digest.py` (feedparser + requests, no Miniflux): `feeds.txt` + `interests.md` live in the inbox repo; `deepseek-chat` scores 1–5 + one-liners; ≈ $0.002/run; laptop `inbox-sync.timer` mirrors into `freelance-practice/_inbox/digest/` for Obsidian | feed fails → skipped; model retried once; else exit ≠ 0 → ntfy + red check | 3 | <1 |
| 2.3 | **Weekly admin triage** — **done 2026-09-13** | Mon 07:00 → `inbox/triage/YYYY-Www.md`; ntfy with top items + `Click` to GitHub | `claude -p` 2.1.270 as `auto` (API key in own Console workspace, $5/mo cap; `--bare`, Read/Glob/Grep only, `--max-budget-usd 1`) over the **root-owned vault mirror** `/srv/vaults/freelance-practice` (hourly pull, read-only deploy key) + latest digest; prompt in inbox `triage/prompt.md`; ≈ 50 s, < $0.10/run | exit ≠ 0 → ntfy + red check; **stale mirror = confident nonsense** — laptop `vault-push.timer` keeps it fed | 2 | <1 |
| 2.4 | **Client research memo** | weekly per active client → `inbox/clients/<name>/YYYY-Www.md` | `claude -p` + web search: what changed in their niche/platforms; template in vault `templates/` | same | 2 | ≈2 |
| 2.5 | **Prospect research** | weekly → `inbox/prospects/YYYY-Www.csv` | Playwright/Crawl4AI: directories, Impressum pages, job boards; **businesses only**, source URL, why; 90-day deletion cron | alert; robots respected; rate-limited | 4 | ≈1 |
| 2.6 | **Own n8n** — **done 2026-09-13** | `https://netcup-vps.tail82bb86.ts.net`, tailnet only | principal `n8n`, Quadlet: `postgres:16-alpine` + `ghcr.io/n8n-io/n8n` 2.38.7 (digest-pinned), podman secrets, **`tailscale serve` instead of Caddy** (Tailscale cert, no host listener); loopback 5678 fenced from `dev`/`auto`/`hermes` (measured); nightly `pg_dump` into the restic set; encryption key in KeePassXC | `n8n-alive.timer` → ntfy + healthchecks; kill switch `systemctl stop user@1004.service` | 1 | 0 |
| 2.7 | **Hour log** — **done 2026-09-13** (redesigned: manual, not tmux — presence ≠ work) | `hours start <project>` / `stop` / `note` on the laptop → `freelance-practice/business/hours.csv`; triage sums it; dictable to the secretary | 20-line shell script | — | 0.2 | 0 |
| 2.8 | **Secretary (Hermes)** — **done 2026-09-13**, [ADR-0036] | Telegram DM ↔ answers from the vault mirror + inbox with file citations; `note:` → `inbox/secretary/` (host timer pushes every 15 min) | Hermes official image, Quadlet under `hermes`, mounts are the boundary, **no port**, no git key inside, DeepSeek; traps: `DropCapability=ALL` kills the init (add SETUID SETGID CHOWN DAC_OVERRIDE FOWNER); `debugging` in `disabled_toolsets` removes `file`; never a second `hermes` process on `/opt/data` | `hermes-alive.timer` hourly → ntfy + healthchecks; kill switch `systemctl stop user@1003.service` | 3 | ≈1–3 |

Total ≈ 20 h; ≈ €15–25/month at DeepSeek/nano prices for the producers plus
whatever the secretary talks. `claude -p` draws from the Max plan today
(help article 15036540 read 2026-09-13: Agent SDK credit paused, `claude -p`
still on subscription limits — so `auto` uses an API key, not a Max token).
Vault access decided and built: root mirrors into `/srv/vaults/`, readers get
the path, `auto` keeps exactly one key.

## 3. Secretary containment spec (2.8)

The reason Hermes is acceptable here and OpenClaw is not, and the reason it is
last: its 2026 CVEs were in file tools and adapters (patched), the malicious
skills were third-party (not installed), and a contained container removes
the host from the blast radius. What remains is the gateway itself and prompt
injection over chat — bounded below.

- **Principal** `hermes`, uid 1003, no SSH, no sudo, egress rules as `auto`.
- **Container**, rootless podman, `--cap-drop=ALL`, `no-new-privileges`,
  image pinned by digest, rebuilt on Hermes releases (their cadence is high —
  a monthly rebuild is the care task).
- **Mounts:** `inbox` clone (rw, own deploy key), vault clones (**ro**, pulled
  by a timer), `hermes` state dir. Nothing from `dev`, nothing from `/etc`.
- **Gateway:** no port published at all (Telegram is outbound polling) — built this way; allowlist of exactly one user ID; no other adapters.
- **Model:** API keys only (DeepSeek for chat, a stronger model for drafting
  if wanted); hard spend caps; **never** the Claude Max or ChatGPT OAuth (ToS).
- **Skills:** none from ClawHub or any registry. Its own tool list limited to
  read files, write under `inbox/secretary/`, run `git` on the inbox, search
  the web. No shell tool exposed to chat. Approval mode on for anything else.
- **Memory:** Hermes's built-in memory stays inside the container state dir; in
  the backup set (it is derived from your data), not credentials.
- **Kill switch:** `systemctl stop user@1003.service` from the admin lane (Quadlet user unit; linger restarts it on `start`).

Acceptance: from Telegram, "what's open on Chantal" answers from the vault
clone; "note: call IHK Tuesday" lands as a file in `inbox/secretary/`; a
message containing "run `rm -rf ~`" produces a refusal and no shell ever
runs; `nc` to the laptop from inside fails.

## 4. Unknowns

| Unknown | Close by |
|---|---|
| `claude -p` billing policy at build time | Anthropic help article 15036540 before 2.3 |
| ~~Hermes: does the current release let the tool list be restricted to the set above without patching~~ | closed 2026-09-13: yes — `platform_toolsets.telegram`; keep `debugging` and `terminal` out of `agent.disabled_toolsets` |
| Hermes memory footprint on 8 GB beside two coding sessions | `systemd-cgtop` in week 1; `user-1003.slice` MemoryHigh 1G / Max 1.5G set; first hours ≈ 260 MB peak |
| Prospect sources that are lawful to crawl and useful for yoga/creator/small-business niches | 2.5 spike: one hour, three sources, note results in `marketing/` (Mentor's folder — route the outreach question there) |
| ~~Whether Miniflux is worth a container vs `feedparser` in the script~~ | closed 2026-09-13: the script is enough |

## 5. Order and stopping points

2.0 ✓ → 2.1 ✓ → 2.2 ✓ (one evening, 2026-09-12/13) → 2.3 ✓ (Sunday morning) (the one the
secretary needs most) → 2.7 (trivial) → 2.8 (secretary; three evenings) →
2.4 → 2.5 → 2.6. n8n last: it is the tool you will sell, but nothing here
needs it yet, and the SUL means it only ever runs *your* workflows.

Each step ends with: unit enabled, one successful run, one forced failure
that reached the phone, a runbook paragraph, a line in ADR-0034's table.
