---
title: "2026-09-12/13 — stage 2 in one weekend: auto, digest, triage, secretary, n8n, Gmail classifier"
type: session
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-12
commit: pending
created: 2026-09-12
updated: 2026-09-13
tags: [session, netcup, automation, systemd, ntfy, git, auto]
---

# 2026-09-12 (evening) — stage 2.0

First stage-2 step, one hour, no transport faults: `auto` has a write-only
path to one repo and nothing else, its jobs run under root-owned units with
the existing ntfy template, and both the success and the failure path were
seen. Runbook `runbooks/automations-stage-2.md` §2.0.

## Decision made on the way

The design note said user units + linger + a user-level `OnFailure` template.
Built instead: **system units with `User=auto`**. Every change to `auto`
already passes through the admin lane (no SSH login), so user units bought
nothing; system units gave three things for free — the root `ntfy-alert@%n`
template, `LoadCredential=` from `/etc/credstore/` for future API keys, and
unit files an agent running as `auto` cannot edit. Linger kept for rootless
podman. Recorded as an amendment line in ADR-0034; the design note's 2.0 row
is superseded by the runbook's conventions table.

Unit pattern for every `auto-*` job: `User=auto Group=auto Slice=auto.slice
OnFailure=ntfy-alert@%n.service NoNewPrivileges PrivateTmp
ProtectSystem=strict ReadWritePaths=/home/auto/inbox`. Scripts root-owned in
`/usr/local/lib/auto/`. `ProtectSystem=strict` cost nothing: git and ssh ran
first time under it.

## Measured

| Check | Result |
|---|---|
| `loginctl show-user auto -p Linger` / `systemctl show auto.slice -p MemoryMax` | `Linger=yes` / `2147483648` |
| `systemctl cat ntfy-alert@.service` | topic-only form, `LoadCredential=ntfy-topic`, `-m 20`, Priority high — the restic runbook's token variant was never deployed; note added there |
| `ssh-keygen -lf ~/.ssh/known_hosts` | matched GitHub's published Ed25519 fingerprint |
| `ssh -T gh-automation-inbox` | `Hi Luis-Gordon/freelance-automation-inbox!`, exit 1 (GitHub's no-shell answer) |
| `git ls-remote` on `freelance-practice` with the same key | `Repository not found`, exit 128 — **the negative test** |
| `nc` github.com 22 / laptop 22 as `auto` | succeeded / timed out — egress rule holds for uid 1002 |
| empty clone + `git symbolic-ref HEAD refs/heads/master` | root commit `818bf09` on `master`, pushed, GitHub default branch set |
| `auto-inbox-selftest.service` | exit 0, 3 s, commit `68ba6f7 selftest: 2026-09-12T15:19:28Z` |
| same unit with the key moved away | exit 1; `ntfy-alert@auto-inbox-selftest.service.service` fired; phone notified within seconds; key restored |

Public key and fingerprint saved to KeePassXC; the private key exists only on
the host (ADR-0027 direction: GitHub is below the VPS).

## 2.1 — heartbeat (past midnight)

healthchecks.io account created (credentials and ping URL in KeePassXC).
First design — `ExecStartPre=…/start`, `ExecStopPost=…/$EXIT_STATUS` — ran
and **half-worked**: the start ping went out, the result ping logged
`hc-ping: no credentials dir`. `ExecStopPost=` runs after systemd has torn the
credentials directory down. Replaced within ten minutes by `hc-run.sh`, a
wrapper that pings start, runs the job, pings the exit status with the job's
output as the POST body. Three runs (0 / forced 1 / 0) all visible in the
check's log. Weekly timer enabled; next fire Mon 06:31. `ProtectSystem=strict`
and `PrivateTmp` did not interfere with `mktemp` or curl.

## 2.2 — trend digest (01:00)

`digest.py` written and unit-tested off-host against a synthetic feed and a
mocked model reply, then shipped by `scp`. Sources now live in
`build/auto/` (new subfolder — the destructive-scripts folder was the nearest
versioned home; a proper repo with the review gate is the right place once the
scripts grow). First dry run under `systemd-run` as `auto` with the real key:
15 s, 60 items, ≈ $0.002, sixteen kept, none scored 5 — honest, nothing in the
daily tools changed that day. `anthropic.com/rss.xml` is a 404 (no feed
exists). Real run committed `d2bbbf4`; forced failure by moving the key file:
the unit fails at credential load, before `ExecStart`, and `OnFailure` still
reaches the phone — good to know, it means a missing credential is loud.
DeepSeek balance $22 at start; at this rate it lasts years.

**Reading it turned out to be the real design question.** First answer — a
laptop user timer mirroring digests into `freelance-practice/_inbox/`
(gitignored) — works but needs the laptop on, and the phone is where the
reading happens. Second answer costs nothing: the ntfy push carries the top
titles and a `Click` to the file on GitHub, which renders it; tap → GitHub app
→ digest, links work. Both kept. `digest.py` v2 also stops a same-day re-run
from overwriting the file with an empty digest.

## Vault mirror + 2.3 (Sunday morning)

Anthropic key created in its own Console workspace with a $5 limit. GitHub's
deploy-key form 500'd seven times; `gh repo deploy-key add` from the laptop
worked first try. Mirror cloned, timer on, negative tests as `auto` all as
expected. Claude Code 2.1.270 installed for `auto`.

**The first triage was confidently wrong — and it was the vault's fault.** It
found no reply window, no meeting, an unsent inventory: the mirror was at the
scaffold commit because nothing had ever pushed the Mentor's writes. Luis
noticed immediately from the content. Committed the backlog, added an hourly
`vault-push.timer` on the laptop (the mirror side already had `inbox-sync`),
refreshed the mirror, reran: a correct five-section triage in 50 s whose top
item was a same-day deadline. Lesson for every reader of a mirror: **check the
mirror's HEAD date before trusting its conclusions** — the triage prompt could
print it in a `## Notes` line; backlog.

`claude -p` under `ProtectSystem=strict` needed only `CLAUDE_CONFIG_DIR` pointed
at the unit's `StateDirectory`; `--bare` skipped hooks/skills/MCP discovery, which
is exactly what an unattended principal should skip. `MultiEdit` is gone as a
tool name.

## 2.7 — hour log (13:00)

Luis killed the tmux-hook design in one sentence: an open ssh session is presence,
not work, and half the work (phone, calls, research, Mentor sessions) never
touches the VPS. Replaced by a manual `hours start|stop|note` on the laptop
writing `business/hours.csv` into the vault; the triage sums it. Ten minutes.

## 2.8 — the secretary (13:00–13:20)

Docs read first: Hermes's per-platform toolsets are config
(`platform_toolsets`), writes can be fenced (`HERMES_WRITE_SAFE_ROOT`), Telegram
is outbound polling — so no port at all, better than "tailnet-only". Three
rounds of running-it-to-find-out: (1) root-owned `~/.config` → podman refused
(`install -d -o` owns only the leaf); (2) `DropCapability=ALL` → s6 init dies at
`s6-applyuidgid`, exit 2, logs gone with `--rm`; foreground run showed it; five
caps added; (3) file tools absent from Telegram although configured — an hour of
source reading here found no coupling; a handoff to ChatGPT found it in a minute:
`debugging` is a composite of terminal+web+file and disabled composites expand to
leaves. Confirmed verbatim in `toolsets.py`. Along the way, running `podman exec
hermes hermes tools …` beside the live gateway corrupted `state.db` — reset
cost nothing, lesson recorded. GitHub's deploy-key web form 500'd again; `gh`
worked. All acceptance tests from the design note passed on the first clean run.
ADR-0036 written. Cost so far today: DeepSeek cents, Anthropic < $0.10.

## Afternoon detour — the inbox (personal, not practice; noted for the classifier)

Gmail `luiswgordon@gmail.com`: 24,457 inbox threads → ~1,000 in two hours, via
sender reports from a subagent over the Gmail connector, source switch-offs
(Facebook, LinkedIn, Arbeitsagentur, Indeed, ImmobilienScout), bulk deletes with
Gmail's "select all conversations that match" (only offered in **Most recent**
sort, not "most relevant"), and the old classifier's `Actions/Delete` +
`Topics/Subscriptions` labels (sampled, then trusted). 188 threads starred by
an agent first — collection letters, official mail, family — so `-is:starred`
could guard every sweep. The Gmail connector's OAuth scope was read-only until
re-linked. Hub decided with the Mentor: `luiswgordon@gmail.com`; business
address `luiswg@neurogenesis.dev` via Cloudflare Email Routing (FDR-004).

## 2.6 — n8n (evening)

Fifth principal, Quadlet, Postgres + n8n 2.38.7, secrets as podman secrets,
`tailscale serve` instead of Caddy (Debian's is 2.6.2 from 2022; `serve` gives a
tailnet-only HTTPS front with a Tailscale cert and no new listener). Docker Hub's
anonymous pull limit bit on the n8n image; `ghcr.io` serves the same digest.
Loopback fence for the other principals measured. Owner account created, dump
timer, alive timer, kill switch → phone, linger restart. About 50 minutes.

## 2.9 — Gmail classifier, n8n's first workflow (23:00–00:00)

Luis's brief: "from experience I know this can be an ordeal." The ordeal was
Google, not n8n. Reused Cloud project `n8n-neurogenesis`: its January client
pointed at the retired `n8n.neurogenesis.dev`; replaced, then deleted and
recreated (a warning flag on the old one, and the secret was gone anyway). The
consent screen's three gates in a row — empty scope list, Branding fields
required before *Publish*, a "branding verification issues" dialog from the
January attempt — each looked like a blocker and none was: client and consent
screen are separate objects, the questionnaire is dormant until you submit,
verification of a restricted scope is a paid CASA assessment nobody runs for a
one-user app. Published unverified; `/about` stands in for the privacy policy
(backlog). "Account connected" on the first sign-in.

Workflow written here as importable JSON (two Code nodes carry the logic, one
Haiku call per run, labels only) plus a two-node error workflow. Diagnostic
before trusting it: *Execute step* on *Get messages* alone and read the field
names — which surfaced the real bug: the node ran once **per label item** (32×)
and tripped Gmail's per-minute quota. *Execute Once* fixed it; the JSON now
carries it. Second stop: n8n 2.x denies `$env` by default — one env line. Then
green end to end: 9 mails, sensible labels on inspection, 3 pushes, 0 drafts.
Luis widened `Official` to account/security notices; prompt updated. Backfill
of the ~1,000 remaining threads started overnight (limit 30, ntfy and draft
nodes off). Anthropic workspace `netcup-vps-n8n`, $5 cap.

## Records touched

- `runbooks/automations-stage-2.md` — new, §2.0–2.3, 2.6, 2.7, 2.8, 2.9 + vault mirror `measured`; `runbooks/_index.md` row.
- `build/n8n/` — Quadlet units, env template (+ `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, `NTFY_TOPIC`), Caddyfile (fallback), dump/alive units, `gmail-classifier.json`, `n8n-error-ntfy.json`.
- `decisions/20260913-contained-secretary-hermes.md` — ADR-0036, accepted; `decisions/_index.md` row.
- `build/hermes/` — Quadlet, config, SOUL, push/alive units; `build/auto/laptop-hours.sh`.
- `build/auto/` — new: all `auto` scripts and units as versioned sources; `build/_index.md` row.
- ADR-0034 — amendment line; healthchecks ping-URL row `2.1 ✓`; DeepSeek key and ntfy-reuse rows `2.2 ✓`; Anthropic key and root mirror key rows `2.3 ✓`; hermes bot token, DeepSeek key, deploy key rows `2.8 ✓`; n8n secrets row `2.6 ✓`; Gmail OAuth grant, Anthropic `netcup-vps-n8n` key, ntfy-in-env rows `2.9 ✓`; `auto` deploy key row `2.0 ✓`; `auto` API key row → `/etc/credstore` + `LoadCredential=`.
- `runbooks/restic-b2-backup.md` — note that the deployed ntfy unit is the topic-only form.
- `sessions/_index.md`, `_index.md` status line.

## Next

2.3 done. (Prerequisites were settled 01:30: help article
15036540 (read 2026-09-13) — the Agent SDK credit is paused, `claude -p` still
draws from the subscription; **decision: `auto` gets an Anthropic API key in
its own Console workspace with a $5/month limit** (`/etc/credstore/anthropic-auto-key`),
not a second Max OAuth token. **Vault access: root mirrors `freelance-practice`
into `/srv/vaults/` with a read-only deploy key on a timer; `auto` (later
`hermes`, ro mount) reads it** — `auto` keeps exactly one key.)

Classifier done (2.9). Next: after the backfill, restore the query/limit and
reactivate ntfy + draft nodes, then the human sweep of `Actions/Delete` and
`Read-Later`; Cloudflare Email Routing + Send-as (FDR-004 evening 3); 2.4 waits
for Chantal's answers; 2.5 parked. Backlog: real Datenschutz page on
neurogenesis.dev → swap the OAuth privacy URL; Android "Open by default" for
`mail.google.com` so ntfy taps open the Gmail app; triage prints the mirror HEAD
date; strip utm from digest links; secretary has no clock (`created:
00:00:00Z`); SOUL line on `[[link|label]]` syntax; monthly Hermes and n8n image
bumps as care tasks. 2.7 hour log is the trivial filler for a short
evening. Before 2.3: re-read Anthropic help article 15036540 on `claude -p`
billing.
