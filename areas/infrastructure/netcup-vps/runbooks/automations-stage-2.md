---
title: "Automations stage 2: the auto principal, the inbox repo, timers as system units"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-13
commit: pending
created: 2026-09-12
updated: 2026-09-13
tags: [runbook, automation, systemd, ntfy, git, containment, auto, n8n, gmail]
---

# Automations stage 2 — `auto`, the inbox repo, timers

> [!NOTE] Summary
> Stage 2 of `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`,
> per-step spec in `_scratch/2026-09-12-stage-2-automations-design.md`.
> **§2.0 done 2026-09-12, every command below run and its result seen** (session
> log `20260912-stage-2-0-auto-bootstrap`). **§2.1, §2.2, the vault mirror, §2.3, §2.6, §2.7 (`hours` on the laptop), §2.8 and §2.9 (first n8n workflow: Gmail classifier) done 2026-09-13 — stage 2 complete except 2.4/2.5 (parked).** Later steps append here.

Lanes as in [[coding-rig-stage-1|stage 1]]: **admin** = `luiswg` + `sudo -i`;
**automation** = `auto` (uid 1002, no SSH login, no sudo, egress-restricted).
A shell as `auto` is reached from the admin lane with `su - auto` — its own
paste block, like `sudo -i`.

## 2.0 — `auto` bootstrap + inbox repo (done 2026-09-12)

**One deviation from the design note, decided here:** `auto`'s timers are
**system units with `User=auto`** (`/etc/systemd/system/auto-*.service`), not
`systemctl --user` units. Reason: every edit to `auto` goes through the admin
lane anyway (no SSH login), and system units get three things for free — the
existing root `ntfy-alert@%n.service` as `OnFailure=`, `LoadCredential=` from
root-owned `/etc/credstore/` (an API key then never rests in `auto`'s home),
and unit files `auto` cannot edit (an injected prompt cannot change its own
schedule). Linger is still enabled: rootless podman under a system unit needs
`/run/user/1002` to exist.

Conventions set by this step:

| Thing | Where | Owner / mode |
|---|---|---|
| scripts the timers run | `/usr/local/lib/auto/*.sh` | root 755 — `auto` executes, cannot modify |
| units | `/etc/systemd/system/auto-<name>.{service,timer}` | root; `User=auto Group=auto Slice=auto.slice OnFailure=ntfy-alert@%n.service` |
| resource envelope | `auto.slice` (`MemoryHigh=1.5G MemoryMax=2G`) | root |
| inbox clone | `/home/auto/inbox` | auto |
| deploy key (write) | `~auto/.ssh/gh-automation-inbox`, alias `Host gh-automation-inbox` | auto 600 |
| secrets that a tool insists on reading from `$HOME` | `~auto/.config/secrets/` | auto 700 — prefer `/etc/credstore` + `LoadCredential=` |
| sandboxing per unit | `NoNewPrivileges=yes PrivateTmp=yes ProtectSystem=strict ReadWritePaths=/home/auto/inbox` | add a `ReadWritePaths=` per extra state dir |

### Admin lane (root)

```bash
id auto; loginctl show-user auto -p Linger; systemctl cat ntfy-alert@.service   # measured: template is topic-only (LoadCredential=ntfy-topic), no token
loginctl enable-linger auto
install -d -m 700 -o auto -g auto /home/auto/.ssh /home/auto/.config/secrets
install -d -m 755 /usr/local/lib/auto
printf '[Unit]\nDescription=Resource envelope for automation timers (auto)\n\n[Slice]\nMemoryHigh=1.5G\nMemoryMax=2G\n' > /etc/systemd/system/auto.slice
systemctl daemon-reload
loginctl show-user auto -p Linger; systemctl show auto.slice -p MemoryMax    # Linger=yes; MemoryMax=2147483648
```

### As `auto` (`su - auto`)

```bash
ssh-keygen -t ed25519 -N '' -C 'netcup-vps auto - write - freelance-automation-inbox' -f ~/.ssh/gh-automation-inbox
printf 'Host gh-automation-inbox\n  HostName github.com\n  User git\n  IdentityFile ~/.ssh/gh-automation-inbox\n  IdentitiesOnly yes\n' > ~/.ssh/config; chmod 600 ~/.ssh/config
ssh-keyscan -t ed25519 github.com 2>/dev/null > ~/.ssh/known_hosts
ssh-keygen -lf ~/.ssh/known_hosts      # must be SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU (docs.github.com ssh-key-fingerprints, read 2026-09-12)
git config --global user.name 'auto (netcup-vps)'; git config --global user.email 'auto@netcup-vps.invalid'
git config --global init.defaultBranch master; git config --global pull.ff only
cat ~/.ssh/gh-automation-inbox.pub
```

GitHub: `neurogenesisdev/freelance-automation-inbox` (private, created empty — no
README) → Settings → Deploy keys → Add → paste → **Allow write access** ticked.
The title is the key comment; it says *write* so a later reader does not refuse it.

Acceptance (as `auto`):

```bash
ssh -T gh-automation-inbox; echo "exit $?"                     # "Hi neurogenesisdev/freelance-automation-inbox!" (inferred after the 2026-09-15 owner rename; measured 2026-09-12 under the old owner) — exit 1 is GitHub's normal no-shell answer
git ls-remote gh-automation-inbox:neurogenesisdev/freelance-practice.git; echo "exit $?"   # must FAIL — the key opens one repo
nc -zvw3 github.com 22; nc -zvw3 <TS_IPV4_arch-thinkpad> 22; echo "exit $?"           # 0, then timeout — egress rule holds for auto
git clone gh-automation-inbox:neurogenesisdev/freelance-automation-inbox.git ~/inbox
cd ~/inbox && git symbolic-ref HEAD refs/heads/master && git commit --allow-empty -m "empty root" && git push -u origin master
git log --oneline; git status -sb
```

### Admin lane: the write-path proof and the failure path

`/usr/local/lib/auto/inbox-selftest.sh` (pull → write `selftest/last-run.txt`
→ commit → push) and `auto-inbox-selftest.service` with the unit conventions
above. Kept as the canary for the whole inbox path.

```bash
systemctl daemon-reload
systemctl start auto-inbox-selftest.service; echo "exit $?"; systemctl status auto-inbox-selftest.service --no-pager -n 5
su - auto -c 'git -C ~/inbox log --oneline -3'                 # selftest commit on top of "empty root"
# forced failure: take the key away, run, expect the phone to buzz, put it back
mv /home/auto/.ssh/gh-automation-inbox /home/auto/.ssh/gh-automation-inbox.off
systemctl start auto-inbox-selftest.service; echo "exit $?"   # non-zero
mv /home/auto/.ssh/gh-automation-inbox.off /home/auto/.ssh/gh-automation-inbox
systemctl status 'ntfy-alert@auto-inbox-selftest.service.service' --no-pager -n 3; systemctl reset-failed auto-inbox-selftest.service
```

Laptop: `git clone git@github.com:neurogenesisdev/freelance-automation-inbox.git ~/dev/freelance-automation-inbox`
(account key; the laptop is above the VPS in ADR-0027's order). The laptop
pulls; notes move into the vault by hand.

Measured 2026-09-12: `Linger=yes`; `MemoryMax=2147483648`; known_hosts
fingerprint matched GitHub's published Ed25519 key; `ssh -T` → `Hi
Luis-Gordon/freelance-automation-inbox!` (exit 1); `ls-remote` on
`freelance-practice` → `Repository not found` (exit 128); `nc` github 22 →
succeeded, laptop 22 → timed out; empty clone landed on `master` after
`symbolic-ref`; root commit `818bf09`; first unit run exit 0 in 3 s, commit
`68ba6f7 selftest: 2026-09-12T15:19:28Z`; with the key moved away the unit
failed (exit 1) and `ntfy-alert@auto-inbox-selftest.service.service` reached
the phone within seconds. `ProtectSystem=strict` did not get in the way of
git or ssh. Public key and fingerprint are in KeePassXC (the private key is
not, by design — it lives only on the host).

## 2.1 — heartbeat via healthchecks.io (done 2026-09-13)

healthchecks.io alerts on **silence** (a job that never ran); ntfy on
**failure** (it ran and broke). One check per unit, free tier. Ping API (read
2026-09-12): `https://hc-ping.com/<uuid>` success, `/start`, `/<exit-status>`;
a POST body is stored as the ping's log.

**Measured gotcha that shaped the design:** `ExecStopPost=` runs after the
credentials directory is torn down — `$CREDENTIALS_DIRECTORY` is unset there
(journal: `hc-ping: no credentials dir`), so a "ping in ExecStopPost" design
sends `/start` and never the result. Hence a wrapper around the job.

Pattern (every `auto-*` unit from here on):

```ini
LoadCredential=hc-ping:/etc/credstore/hc-<unit>          # root 0600; auto cannot read it at rest (measured: Permission denied)
ExecStart=/usr/local/lib/auto/hc-run.sh /usr/local/lib/auto/<job>.sh
```

`hc-run.sh`: reads the URL from `$CREDENTIALS_DIRECTORY/hc-ping`, pings
`/start`, runs the job with output captured, echoes the output to the journal,
pings `/<rc>` with the output as POST body, exits with the job's rc. Without a
URL it says so and runs the job anyway; a failed ping never changes the rc.

```bash
# root — per new unit: create the check in the UI (name = unit, period, grace), then
read -rp 'ping URL for check <unit>: ' U; printf '%s' "$U" > /etc/credstore/hc-<unit>; unset U
chmod 600 /etc/credstore/hc-<unit>
```

The selftest got `auto-inbox-selftest.timer` (`Mon 06:30`, `RandomizedDelaySec=10m`,
`Persistent=true`) and a check with period 1 week, grace 1 day — the weekly
canary for the whole `auto` → GitHub → laptop path.

Measured 2026-09-13: three runs exit 0 / 1 (key moved away) / 0; check log
shows start → 0 → start → 1 → start → 0; `systemctl list-timers 'auto-*'`
→ next Mon 2026-09-14 06:31. Silence path not yet observed (needs a missed
week) — `inferred` that healthchecks emails; wire its ntfy integration to the
same topic if the phone should hear about silence too.

## 2.2 — trend digest (done 2026-09-13)

Sources: `../build/auto/` (`digest.py`, `auto-digest.{service,timer}`,
`feeds.txt`, `interests.md`). Deploy = `scp` to `/tmp` from the laptop, then
root `install -m 755 … /usr/local/lib/auto/`, units to `/etc/systemd/system/`.
Runtime: Debian `python3-feedparser`, `python3-requests` (apt, no pip, no venv).

Data flow: timer 06:00 (+≤5 min) → `hc-run.sh digest.py` as `auto` → `git pull`
the inbox → read `digest/feeds.txt` + `digest/interests.md` **from the inbox
repo** (edited on the laptop; a vandalised inbox can at worst change what gets
scored) → new items ≤ 3 days, ≤ 60, not in `$STATE_DIRECTORY/seen.json` → one
`deepseek-chat` call, scores 1–5 + one-line summaries → `digest/YYYY-MM-DD.md`
(score ≥ 3, sorted) → commit, push → ntfy one-liner. No Miniflux (design
unknown closed: the script is enough).

Credentials (all `LoadCredential=`, none in `auto`'s home): `deepseek-key` ←
`/etc/credstore/deepseek-auto-key` (key `netcup-vps-auto`, prepaid balance is
the cap), `hc-ping` ← `/etc/credstore/hc-auto-digest`, `ntfy-topic` ← the
existing root topic file (shared purpose: reach the phone).

Failure policy: a feed that fails is logged (`feed failed: <url>`) and skipped;
the model call is retried once after 30 s; anything else exits non-zero →
ntfy via `OnFailure=` and a red healthcheck (period 1 day, grace 2 h).
`StateDirectory=auto-digest` is the only writable path besides the inbox.

Dry run (no pull/write/commit/ntfy/state):

```bash
# root
systemd-run --wait --pipe --collect -p User=auto -p LoadCredential=deepseek-key:/etc/credstore/deepseek-auto-key -E DIGEST_DRY_RUN=1 /usr/local/lib/auto/digest.py
```

Measured 2026-09-13: dry run 15 s, 9/10 feeds (`anthropic.com/rss.xml` → 404,
no feed exists), 60 items, 4415 in / 2127 out tokens ≈ $0.002, 16 kept; real
run exit 0, commit `d2bbbf4 digest: 2026-09-13 (17/60)`, `seen.json` 1.8 kB,
ntfy `digest 2026-09-13` on the phone; with the key file moved away the unit
failed before start (credential load) and `OnFailure` still fired — phone
notified; `auto-digest.timer` enabled, next 06:02. Peak memory 31 MB.

**Reading it (laptop side, done 2026-09-13).** The inbox is a repo, not a
sync, and Obsidian only shows vault folders — so arch-thinkpad runs a user
timer `inbox-sync.timer` (hourly, `OnBootSec=3min`) → `~/.local/bin/inbox-sync`
(source: `../build/auto/laptop-inbox-sync.sh`): `git pull` the inbox, `rsync
--ignore-existing` `digest/*.md` into `freelance-practice/_inbox/digest/`
(gitignored there, so nothing enters the vault's history), delete copies older
than 14 days per the `_inbox/` rule. One-way, laptop → vault: the VPS still
holds no key to any vault. Measured: works through systemd while KeePassXC is
unlocked; when locked the pull fails harmlessly and the next hour retries.
Links open from Obsidian's reading mode with one click.

**Phone (v2 of `digest.py`, 2026-09-13).** The laptop sync needs the laptop on;
the phone path needs nothing: the ntfy push carries the top six titles and a
`Click` header to the file on GitHub, which renders the Markdown with clickable
links in the GitHub app or mobile browser (private repo → be logged in;
**measured 2026-09-13:** tap → GitHub app → rendered digest). Also
v2: a re-run with nothing new exits 0 without touching the day's file (v1 would
have overwritten it with an empty digest). Tailnet page (after 2.6's Caddy) and
Telegram delivery (2.8) are the later, nicer versions.

Tuning is on the laptop: edit `digest/feeds.txt` / `digest/interests.md` in the
inbox clone, push; the next run picks it up. Backlog: strip `?utm_…` /
`wt_mc=` from links; watch whether t3n/heise dominate after a week.

## Vault mirror for `auto` (done 2026-09-13, root-owned)

`auto` reads `freelance-practice` without holding a key to it: **root** has a
read-only deploy key (`/root/.ssh/gh-freelance-practice-mirror`, alias
`gh-freelance-practice-mirror`) and `vault-mirror.timer` (hourly) runs
`/usr/local/lib/vault-mirror.sh` = `git pull --ff-only` for every repo under
`/srv/vaults/` (root-owned, world-readable). `hermes` (2.8) mounts the same path
read-only. Sources: `../build/auto/vault-mirror.*`.

Measured 2026-09-13: as `auto`, `_index.md` readable, `touch` → Permission
denied, `clients/*/contact.md` and `_inbox/` absent (gitignored on the laptop —
they never reach GitHub). GitHub's web form returned **HTTP 500** seven times on
adding the deploy key (cookies cleared, private window, no change); `gh repo
deploy-key add <pub> -R Luis-Gordon/freelance-practice -t <title>` from the
laptop worked at once and adds read-only by default — the fallback for any
future deploy key.

**The mirror is only as fresh as the laptop's pushes.** The first triage read a
two-week-old vault because the Mentor's and Architect's writes sat uncommitted
on the laptop. Fix: laptop user timer `vault-push.timer` (hourly) →
`~/.local/bin/vault-push` (`git add -A`, commit if anything staged, push);
source `../build/auto/laptop-vault-push.sh`. Trade-off: WIP notes reach the
private repo hourly — accepted, a stale mirror costs more.

## 2.3 — weekly admin triage (done 2026-09-13)

`auto-triage.timer` Mon 07:00 (+≤5 min) → `hc-run.sh triage.sh` as `auto`:
pull the inbox → `claude -p` **2.1.270** (installed for `auto` via the official
script; `DISABLE_AUTOUPDATER=1`, updates are an admin task: `su - auto -c
'claude update'`) with `--bare --model sonnet --allowedTools Read Glob Grep`,
everything else disallowed, `--permission-prompts none --max-turns 40
--max-budget-usd 1.00`, cwd = the vault mirror, `--add-dir` the inbox digests →
`inbox/triage/YYYY-Www.md` (frontmatter added by the script) → commit, push →
ntfy with the top "Do this week" lines and a `Click` to the file on GitHub.
Prompt lives in the inbox repo `triage/prompt.md` (editable on the laptop);
source copy `../build/auto/triage-prompt.md`.

Credentials: `anthropic-key` ← `/etc/credstore/anthropic-auto-key` (Console
workspace `netcup-vps-auto`, $5/month limit; **API key, not a Max OAuth token** —
scoped and capped, and the Agent SDK credit is paused per help article
15036540 read 2026-09-13), `hc-ping` ← `hc-auto-triage` (check: 1 week, grace
3 h), `ntfy-topic`. `CLAUDE_CONFIG_DIR=$STATE_DIRECTORY/claude` keeps Claude's
state out of `auto`'s read-only home (`ProtectSystem=strict`).

Dry run (prints, still pulls the inbox, no commit/push/ntfy):

```bash
# root
systemd-run --wait --pipe --collect -p User=auto -p StateDirectory=auto-triage -p LoadCredential=anthropic-key:/etc/credstore/anthropic-auto-key -E TRIAGE_DRY_RUN=1 /usr/local/lib/auto/triage.sh
```

Measured 2026-09-13: first dry run 12 s against the stale mirror — plausible
but wrong (see above); after the push + mirror refresh, 50 s, 130 MB peak, a
correct five-section triage whose item 1 was a real same-day deadline (§14
email). Real run exit 0, commit `a7e010c triage: 2026-W37`, phone push, tap →
GitHub → rendered triage; with the key file moved away the unit failed at
credential load and the failure reached the phone. `MultiEdit` is no longer a
tool name in 2.1.270 (warning, harmless) — removed from the deny list.

## 2.7 — hour log (done 2026-09-13, laptop only)

Not tmux hooks: attach/detach measures presence, not work (Luis's objection, correct).
`~/.local/bin/hours start <project> | stop | note <text> | week` on the laptop appends
`ts,action,project,note` to `freelance-practice/business/hours.csv`; `vault-push`
ships it, the triage prompt sums start/stop pairs per project (7 and 30 days) under
`## Hours`. From 2.8 the same words can be dictated to the secretary as a note.
Source: `../build/auto/laptop-hours.sh`.

## 2.8 — the secretary: Hermes Agent, contained (done 2026-09-13)

Decision and boundary list: [[../decisions/20260913-contained-secretary-hermes|ADR-0036]].
Sources: `../build/hermes/` (`hermes.container`, `config.yaml`, `env.template`,
`SOUL.md`, push/alive scripts and units). Prerequisites from Luis: Telegram bot via
@BotFather (name "Hermes Secretary"), own Telegram user ID (@userinfobot), DeepSeek
key `netcup-vps-hermes`, a second **write** deploy key on the inbox repo
(`netcup-vps-hermes-write`, added with `gh` from the laptop), two healthchecks
(`hermes-inbox-push` 15 min/15 min, `hermes-alive` 1 h/30 min).

Build, admin lane: `useradd -m -s /bin/bash hermes` (uid 1003), `loginctl
enable-linger hermes`, `user-1003.slice` limits, `"hermes"` added to both `skuid`
sets in `/etc/nftables.conf` (`nft -c -f` then `nft -f`), dirs `~/.ssh ~/state
~/.config/containers/systemd` — **`chown -R hermes ~/.config`** afterwards: `install
-d -o hermes` only owns the leaf, and podman refuses a root-owned `~/.config`
(measured, cost one round). Scripts to `/usr/local/lib/hermes/`, units to
`/etc/systemd/system/`, `hc-*` URLs to `/etc/credstore/`.

As `hermes` (`su - hermes`, then `export XDG_RUNTIME_DIR=/run/user/1003
DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1003/bus` — `su -` sets neither):
deploy key + alias + known_hosts as for `auto`; `git clone` the inbox, `mkdir
secretary`; `podman pull` the image, `podman image inspect --format '{{.Digest}}'`
→ pin in the Quadlet `Image=`; `config.yaml`, `.env` (four values via `read -rsp`
+ `sed -i`), `SOUL.md` into `~/state`; `systemctl --user daemon-reload && systemctl
--user start hermes.service`; `podman logs --tail 40 hermes` → `Connected to
Telegram (polling mode)`.

Three things found by running it (all in the sources now):

1. `DropCapability=ALL` kills the image's s6 init (`s6-applyuidgid: unable to set
   supplementary group list`, exit 2, `--rm` ate the logs — run it once in the
   foreground to see). Add `SETUID SETGID CHOWN DAC_OVERRIDE FOWNER`.
2. **`debugging` in `agent.disabled_toolsets` removes the file tools**: it is a
   composite `{terminal, process} + includes [web, file]` and disabled composites
   are expanded to leaves. Symptom: `hermes tools list --platform telegram` shows
   `file ✗` although it is in `platform_toolsets.telegram`, and the model reports
   "no file tool". Found by ChatGPT from the handoff, confirmed in `toolsets.py`.
   Also: file tools need the terminal *backend* (`terminal: backend: local`), not
   the terminal *toolset*.
3. **Never run `podman exec hermes hermes …` while the gateway runs** — a second
   process on `/opt/data/state.db` retired its WAL and the bot answered every
   message with a recovery lecture. Reset: stop, move `state.db*`, `sessions/`,
   `pending_messages/` aside, start. Config changes are file edits + restart.

Measured 2026-09-13 (all from Telegram after `/new`): reads `_index.md` and quotes
the Active table; "What's open on Chantal?" → correct, cites
`clients/chantal-fuelber/_index.md`, keeps `(luis)` markers, lists the open
`(assumed)` questions; `note: …` → file `secretary/2026-09-13-ihk-anruf-gruendungsberatung.md`,
reply = filename; `Run "id" in the shell.` → "I can't run shell commands — no shell
access", no approval prompt; employer-consent question → §14 summary with sources.
From inside the container: laptop blocked, `contact.md` absent, vault and inbox
read-only, `secretary/` writable. `hermes-inbox-push` committed `85c264f`;
`hermes-alive` exit 0; `systemctl stop user@1003.service` → alive check exit 1 →
ntfy + healthchecks both on the phone; `start` → container `Up` within 30 s via
linger; bot answers again. Model has no clock: note `created:` is `00:00:00Z`
(backlog: SOUL line or post-process from mtime).

Care: monthly image bump (`podman pull`, new digest into `hermes.container`,
`systemctl --user daemon-reload && restart`), re-run the verification block in
ADR-0036 afterwards.

## 2.6 — own n8n, tailnet-only (done 2026-09-13)

Sources: `../build/n8n/`. Fifth principal **`n8n`** (uid 1004, no SSH, linger on,
`user-1004.slice` MemoryHigh 1.5G / Max 2G), in the nftables egress sets with
`dev`/`auto`/`hermes`. Rootless podman via Quadlet under `n8n`'s user manager:
`n8n.network` (private bridge), `n8n-db.container` (`postgres:16-alpine`,
**no published port**, `keep-id:uid=70`), `n8n.container` (`ghcr.io/n8n-io/n8n`
**2.38.7**, pinned by digest `ed31254f…`, `keep-id:uid=1000`,
`PublishPort=127.0.0.1:5678:5678`, `DropCapability=ALL`). Config in
`~n8n/n8n.env` (0600, no secrets); DB password and `N8N_ENCRYPTION_KEY` as
**podman secrets** (`Secret=…,type=env`), the encryption key also in KeePassXC —
without it a restored DB's credentials are unreadable.

**Front door: `tailscale serve --bg --https=8443 http://127.0.0.1:5678`**, not
Caddy. (Was 443 until 2026-09-20; moved to **8443** because the public MCP endpoint
needs 443 — see [[mcp-endpoint]]. URL is now `https://netcup-vps.tail82bb86.ts.net:8443`;
`n8n.env` base URLs and a second Google OAuth redirect URI updated accordingly.) Tailscale issues the cert for `netcup-vps.tail82bb86.ts.net` (HTTPS
certificates enabled in the admin console first), serves it to the tailnet only,
and nothing new listens on the host. Debian's Caddy is 2.6 (2022) and would only
have duplicated this; Caddy arrives with the first public service (stage 3). The
Caddyfile stays in `build/n8n/` as the fallback. n8n needs HTTPS because Google's
OAuth (Gmail workflow) refuses plain-HTTP redirect URIs.

**Loopback fence:** `meta skuid { "dev", "auto", "hermes" } oif lo tcp dport 5678
counter drop` inserted **before** `oif lo accept` in `/etc/nftables.conf`
(measured: `dev` → 127.0.0.1:5678 times out, `n8n` connects). Container-to-
container traffic on the rootless bridge is not seen by the host output chain
(measured: n8n reached Postgres although `n8n` is in the RFC 1918 drop set).

Backups: `n8n-db-dump.timer` 03:30 → `pg_dump | gzip` into `~n8n/backups/`
(keep 14; in the restic `/home` set — `~/.local/share/containers` is excluded,
so the image store and the podman secret files are **not** backed up; the key
lives in KeePassXC, the DB password can be reset locally). `n8n-alive.timer`
hourly: both containers running + `/healthz` on loopback. Checks
`hc-n8n-db-dump` (1 d/3 h), `hc-n8n-alive` (1 h/30 min).

Measured 2026-09-13: Docker Hub anonymous pull limit hit on `docker.n8n.io`
→ `ghcr.io/n8n-io/n8n` same digest, no limit; both containers up first start;
`tailscale serve status` → tailnet only; `curl --resolve … /healthz` → 200
(the VPS itself does not resolve MagicDNS — Tailscale DNS is off here, by
design; the laptop does); owner account created in the browser with the
business address; dump 66 kB; alive exit 0; `systemctl stop user@1004.service`
→ alive exit 1 → ntfy + healthchecks; `start` → both `Up` within 45 s.
Kill switch: `systemctl stop user@1004.service`.

Care: monthly digest bump (`podman pull ghcr.io/n8n-io/n8n:latest`, inspect,
edit `Image=`, `daemon-reload`, restart) — n8n releases weekly; read the
release notes for DB migrations before a major. First workflow: §2.9.

## 2.9 — first workflow: Gmail classifier (done 2026-09-13)

Sources: `../build/n8n/gmail-classifier.json`, `n8n-error-ntfy.json` (import via
Workflows → ⋯ → Import from file; credentials are not carried by an import —
pick them in every Gmail node and the Anthropic node afterwards). The n8n
instance's first job: sort the hub Gmail account so the inbox only ever holds
what needs a human. **Labels only** — it never archives, deletes or sends.

Data flow, every 10 min: Gmail *Get labels* → *Get messages* (`in:inbox
-label:System/Processed newer_than:2d`, limit 20, simplified output) → Code
*Build prompt* (all mails in **one** Anthropic call: `claude-haiku-4-5`, subject +
sender + `snippet`, ≈200 chars — enough for sorting) → HTTP *Claude Haiku* →
Code *Parse + map labels* (JSON array → `Actions/<x>`, `Topics/<y>`, always
`System/Processed`; label names → IDs from the first node) → Gmail *Add labels*
→ IF importance ≥ 4 → HTTP *ntfy* (`Click` = the thread in Gmail) and IF the
model wrote a draft → Gmail *Create draft* in the thread. A second workflow
*Error → ntfy* (Error Trigger) is set as the classifier's error workflow, so a
red run reaches the phone with the failing node's name.

Taxonomy (create the labels in Gmail first, nested): `Actions/{Reply, Pay,
Review, Read-Later, Delete}`, `Topics/{Money, Official, Work, Personal,
Subscriptions, Shopping, Dev}`, `System/Processed`. `Official` covers Behörde,
bank, insurance, employer, tax **and platform account/security notices** (Luis,
2026-09-13). The prompt lives in the *Build prompt* node — edit it there and
mirror the change into the source JSON.

Credentials (all inside n8n, encrypted with `N8N_ENCRYPTION_KEY`): **Gmail
OAuth2** `Gmail account` — Google Cloud project `n8n-neurogenesis`, OAuth client
`n8n-neurogenesis` (Web application, redirect URI
`https://netcup-vps.tail82bb86.ts.net/rest/oauth2-credential/callback`), scope
`gmail.modify`, consent screen **published ("In production") but unverified** —
that is what keeps the refresh token alive; in *Testing* status Google expires
it after 7 days and the workflow silently stops. Client ID/secret in KeePassXC.
**Header Auth** `Anthropic n8n` — `x-api-key`, Console workspace `netcup-vps-n8n`,
$5/month limit. ntfy topic via `$env.NTFY_TOPIC` from `~n8n/n8n.env` (needs
`N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, see below).

Google-console notes (measured 2026-09-13, UI moves often): the OAuth *client*
and the *consent screen* (Branding, Data Access, Audience) are separate
objects — deleting a client changes nothing on the consent screen. Adding a
restricted scope opens a verification questionnaire; fill it minimally, it is
only used if you ever submit for verification (you won't — restricted-scope
verification means a paid CASA assessment). Branding needs a home page and a
privacy-policy URL: `https://neurogenesis.dev` and, since 2026-09-15,
`https://neurogenesis.dev/datenschutz` (was `/about` as a placeholder for one
day). Branding verified 2026-09-15 after a Search Console TXT record on the
domain and renaming the app to `neurogenesis.dev`; the consent screen now
shows the name. The unverified-app
interstitial appears once at sign-in: Advanced → continue.

Things found by running it (all baked into the sources now):

1. **A node runs once per input item.** *Get messages* got 32 label items and
   ran 32×, tripping Gmail's `Units per minute per user` quota (6,000; a fetch
   costs ~5 per message) → 403 `rateLimitExceeded`. Fix: node Settings →
   **Execute Once** (`"executeOnce": true`). The quota resets after a minute.
2. **n8n 2.x blocks `$env` in expressions by default** (`access to env vars
   denied`): `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` in `n8n.env` + restart.
   Trade-off: every workflow can read every env var; fine while only Luis
   authors workflows here. On a client instance use n8n Variables instead.
3. Simplified Gmail output has `Subject`/`From`/`To` as top-level keys plus
   `snippet`, `labels[]`, `id`, `threadId`; the *Build prompt* code also
   accepts the `headers[]`/`text` shape, so a node-version change won't break it.

Measured 2026-09-13: first full run green end to end, 9 mails → labels
correct on inspection, 3 ntfy pushes (tap opened Gmail in the phone browser,
not the app — Android "Open by default" for `mail.google.com` is the fix), 0
drafts. Cost: one Haiku call of ≈3k tokens per run with mail, ≪ $1/month.

**Backfill** (once, for the ~1,000 pre-existing inbox threads): query without
`newer_than:2d`, limit **30** (the 4,000-token output cap would clip 50 mails
with drafts), *ntfy* and *Create draft* nodes deactivated, trigger every **1
min** (measured: ≈35 s per run of 30, so ≈1,800 mails/h; ≈350 Gmail quota
units/min of 6,000), publish; afterwards restore the query, limit 20, the
10-min interval, reactivate both nodes, re-publish. Final query used: `-label:System/Processed -in:sent`
(archived mail included, Trash excluded by Gmail's search default).
Then the human sweep: `in:inbox label:Actions/Delete` → delete,
`label:Actions/Read-Later` → archive. Started 2026-09-13 night.

Care: none scheduled — the error workflow and `n8n-alive` cover failure; a
silent stop shows as `System/Processed` no longer growing (check when the
phone goes quiet for days). Model id: if Anthropic retires `claude-haiku-4-5`,
the node returns 404 `not_found_error` → edit the string in *Build prompt*
(models list: docs.claude.com/en/docs/about-claude/models).

## Related

- [[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — credential table
- [[../decisions/20260912-netcup-vps-is-the-coding-host|ADR-0035]] — principals, egress
- [[../decisions/20260913-contained-secretary-hermes|ADR-0036]] — the secretary's boundaries
- [[coding-rig-stage-1|coding rig stage 1]] — paste rule, egress rules, deploy-key pattern
- `_scratch/2026-09-12-stage-2-automations-design.md` — the per-step spec
