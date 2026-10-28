---
type: design
status: proposed
provenance: inferred
device: netcup-vps, arch-thinkpad
created: 2026-09-10
updated: 2026-09-11
tags: [devenv, agents, containment, automation, tailscale, backups]
---

> [!NOTE] Revision 2026-09-11
> Reviewed by a second model (GPT). Adopted: per-harness deny rules (the Unix
> user is not the agent boundary), sandbox fail-closed, ADR-0021 rewrite moved
> to step 0.1, per-repo deploy keys, `authorized_keys` step, symlink path,
> `nc`-based firewall test with counters, loopback inventory, automation
> write-path for the vault, backup deletion-resistance, n8n as its own
> principal, §7 UWG wording, Codex quota numbers removed. Rejected after
> checking the docs: "IdentityFile .pub is a bug" (your key lives in the
> KeePassXC agent; `.pub` + `IdentitiesOnly` is the documented way to select
> it), "cross-session messaging is stale" (current docs, and they state
> containers break it), "`claude -p` bills to a separate credit since June"
> (announced May, paused June — still an unknown). Agent View added as
> supervisor for long Claude jobs, not as replacement for tmux.
>
> **Revision 2 (same day), after a second GPT pass.** Adopted: containment is
> heterogeneous per harness (Claude `sandbox.credentials`/`denyRead`; Codex
> permission profiles; OpenCode in the devbox container because its `shell`
> permission "runs with the host user's … authority"); invariant "agents
> commit, humans push/deploy"; `--add-dir` + `denyWrite` for context vaults;
> `auto` gets a separate inbox repo and no vault key; positive B2 capability
> list; nftables `ct state` + 169.254/16; unknown #3 rewritten as a known
> volatile policy. Rejected: a separate Unix user for OpenCode (shared-worktree
> ownership friction; the container is the better primitive and is already
> your recipe). Noted for later: OpenCode has no peer-session messaging today,
> so containing it costs nothing; if it gains one, moving it onto the host is
> the revisit condition.

# AI coding rig + automation platform — decision record and staged plan

> [!NOTE] Summary
> The netcup VPS becomes the one authoritative development host (worktrees
> mean several working directories, one host; tmux; sessions survive the laptop). Three harnesses, each on its own legal
> credential: Claude Code on Max (primary coder), Codex CLI on ChatGPT
> (independent reviewer), OpenCode on API keys (any-model harness — it *is* the
> router). No router in front of Claude Code. Orchestration is native Claude
> Code (worktrees, subagents, cross-session messaging) plus the official
> Codex review plugin — no third-party orchestrator. Automations come in
> stage 2 as systemd timers (`claude -p` for judgment, Python for
> deterministic), ntfy alerts, n8n only for your own instance. Hermes and
> OpenClaw stay off this server. **Backups on the VPS come first — your own
> backlog says nothing there is backed up.**
>
> Provenance: everything below is `inferred` from vendor docs read 2026-09-10
> (sources at the end). Nothing has been run on either machine. Every version
> and price carries the command or URL to re-verify.

## 0. Decisions at a glance

| # | Decision | Rejected | Why |
|---|---|---|---|
| D1 | **VPS is the workstation.** Code lives only on netcup-vps; laptop and phone are thin clients over the tailnet. | Laptop as workhorse + VPS relay; mirrored devbox on both | You chose it; it removes the sync problem entirely and survives laptop sleep. Cost: OAuth tokens on an unencrypted disk (ADR-0021 residual) — accepted with mitigations, see D4. |
| D2 | **Three harnesses, three credentials.** Claude Code ← Max OAuth. Codex CLI ← ChatGPT OAuth. OpenCode ← API keys (Anthropic/OpenAI/DeepSeek) and optionally OpenCode Go. | Max OAuth inside OpenCode/Hermes/OpenClaw | Anthropic's legal page: OAuth is "exclusively … for native Anthropic applications"; third parties "may not … intermediate Claude.ai credentials." OpenCode removed the login in v1.3.0. Not a grey area. |
| D3 | **No router in front of Claude Code.** OpenCode is the multi-model harness; if one key for many models is wanted, OpenRouter behind OpenCode. | claude-code-router, LiteLLM proxy, Requesty, Portkey | Anthropic "doesn't support routing Claude Code to non-Claude models through any gateway"; a gateway credential also replaces your Max plan. LiteLLM is a service to run with no consumer but you. Revisit when two machines share keys. |
| D4 | **Containment is heterogeneous — each harness gets the strongest boundary it actually supports, all under principal `dev` for coding.** Claude Code: on the host, OS sandbox fail-closed, `sandbox.credentials` + `filesystem.denyRead` as the mechanism, `permissions.deny` as UI policy. Codex: on the host, `default_permissions` profiles `reviewer` (read-only, no secrets, no network) and `builder`. OpenCode: **in the devbox container** (rootless podman as `dev`, only the assigned worktree + its own state dir mounted). Principals `auto` (timers) and `n8n` (service) are separate users with no access to `dev`'s home. nftables egress keyed on `skuid` blocks RFC 1918, 169.254/16 and 100.64/10 for `dev`/`auto`. Admin lane = your sudo user. | One `dev` user + 0600 as the boundary; a separate Unix user for OpenCode; containers for every harness | A Unix user does nothing against an agent running *as* that user, so the boundary is drawn where each harness can enforce it at kernel level. OpenCode's own docs: "`shell` runs with the host user's filesystem, process, and network authority … Prefer a narrow shell allowlist" — command-string mediation is not a filesystem boundary, so it gets the container. A separate user for OpenCode would need shared ownership of every worktree (groups/ACLs) — permanent friction for the same result. Claude stays on the host because its cross-session messaging docs state a container "has its own filesystem, so a session inside it and a session on the host can't reach each other." Containing OpenCode costs nothing today: it has subagents with per-agent `model`, but no peer-session messaging. **Revisit when** OpenCode gains peer messaging worth having across harnesses — then it moves onto the host with a Claude-grade sandbox or stays contained and talks via its server API. |
| D4a | **Invariant: agents edit, test and commit in a worktree; a human pushes, merges and deploys.** Deploy keys and `~/.config/secrets/*.env` are `deny` for sandboxed commands; the `!` shell prefix inside a Claude session runs outside the sandbox with your access, so `!git push` is the human step in the same window. Autonomous deploy later = `sandbox.credentials` `"mode": "mask"` scoped to `api.cloudflare.com`, never a weakened sandbox. | Agents holding push/deploy authority in stage 1 | The review gate is a merge gate; giving the coder the key to bypass it makes the gate decorative. Masking is the designed narrow capability when you want it. |
| D5 | **Orchestration = native, layered.** tmux = durable terminal substrate for all three harnesses; git worktrees = mutation boundary (`claude --worktree`); subagents = intra-task delegation (OpenCode: per-agent `model`, e.g. cheap builder + GPT reviewer inside one session); cross-session `SendMessage` = peer coordination between Claude sessions; `claude --bg` / `claude agents` (Agent View) = supervisor for long Claude-only jobs; `codex-plugin-cc` = review gate; `ralph-loop` = bounded grind. Cross-harness handoff = git (worktree, commit, review), or later a script posting to OpenCode's `/session/:id/prompt_async`. | Agent teams as default (experimental, no automatic worktree isolation), Agent View as the *only* control plane (research preview, Claude-only, idle stop ≈ 1 h), Gas Town/Beads, Ruflo, Paperclip, Symphony, Vibe Kanban (shut Apr 2026), Tmux-Orchestrator | Each layer gets one job. Codex and OpenCode do not live in Agent View, so tmux stays underneath. Claude Squad / ccmanager worth trying if you routinely run >2 parallel sessions. |
| D6 | **Terminal layer = tmux + SSH over Tailscale; phone = Claude Remote Control first, Termux second.** | zellij, mosh-first, Eternal Terminal, VS Code Tunnels | tmux is what every guide, plugin and orchestrator assumes. ET is dormant (2023). Remote Control needs no SSH client on the phone. Tailscale SSH stays an open backlog item (ADR-0029 boot-order argument) — plain sshd over the tailnet is enough now. |
| D7 | **Automations (stage 2): systemd timers.** Deterministic → Python; judgment → `claude -p`; alerts → ntfy; n8n self-hosted for *your* workflows only; client n8n in the client's account. | Hermes Agent, OpenClaw on this VPS; Routines (cloud, no local files) | OpenClaw: Jan–Feb 2026 RCE + 12 % malicious skills; Hermes: three CVEs May 2026 and Max-OAuth billing bugs. Neither belongs on the box that holds your vault clones. n8n's Sustainable Use License forbids hosting several clients' workflows on one instance you run. |
| D8 | **Lead generation = research, not outreach.** Crawl public *business* signals; store company, URL, reason; contact by letter, warm intro, public engagement; telephone only where the facts support *mutmaßliche Einwilligung* (§7 Abs. 2 Nr. 1 UWG, B2B). | Instagram scraping, cold DMs/emails | §7 Abs. 2 Nr. 2 UWG: email and social DMs need prior express consent, B2B too (AG Düsseldorf 23 C 120/25). Instagram scraping breaks Meta ToS. Abmahnung risk is real money. Commercial side → Freelance Business Mentor. |

**Incremental cost:** €0–10/month on top of what you pay (OpenCode Go $10 optional; cheap API models ≈ $5–15 at digest volumes; B2 for the VPS ≈ $0.10). The real cost line is a ChatGPT Pro upgrade if Plus's Codex quota (varies by model and has a weekly cap — check the current help-center table) proves too small for review-heavy days — measure two weeks, record the review model you settle on, then decide.

## 1. Constraints this plan must respect (from the vaults)

- **ADR-0027** — the VPS holds no credential that reaches upward. Anthropic/OpenAI tokens, a GitHub deploy key, a scoped Cloudflare token: all "below". A key authorised on arch-thinkpad or pixel-graphene: never. Laptop → VPS only. Compliant.
- **ADR-0021** — no LUKS; residual obligation "no long-lived secret reaches disk in plaintext." This design violates it not once but structurally: OAuth tokens, API keys, restic/B2 key, Cloudflare token, ntfy token, later the n8n encryption key. An unattended, unencrypted server always holds *some* recoverable bootstrap secret, so the property as written cannot be enforced. **Step 0.1 rewrites it into one that can:** every persistent VPS credential is (a) downward-only in the trust order (ADR-0027), (b) independently revocable from the issuing service, (c) least-privilege and scoped to one purpose, (d) owned by exactly one principal and unreadable by the others, (e) listed by name in the vault. Same amendment ADR, one record.
- **ADR-0033** — two lanes. Preserved: coding as `dev`, admin as sudo user. The container half applies to OpenCode now (D4) and to the rest only if the host-side sandboxes prove too thin.
- **FDR-001 / FDR-002** — no client hosting on personal infra; client pays third parties. Coding on the VPS is fine; deploys go to the client's Cloudflare Pages via `wrangler` with a **project-scoped** token, rotated at handover.
- **ADR-0017** — harness state: derived state backed up, credentials and cache excluded. Same split on the VPS (stage 1.8).
- **netcup backlog `restic-client` (high)** — nothing on the VPS is backed up. Stage 0, non-negotiable before daily use.

## 2. Target shape

```
arch-thinkpad (Obsidian vaults, Claude Desktop, KeePassXC)   pixel-graphene (Termux, Claude app)
        │ ssh over tailnet, tmux attach                              │ Remote Control / ssh+mosh
        ▼                                                            ▼
netcup-vps ── admin lane: <sudo-user>   (root tasks, restic, nftables, podman later; restic key readable by root only)
           ├─ coding lane: dev          (no sudo; egress: internet yes, RFC1918+tailnet no; harness deny rules)
           │     tmux "main"
           │     ├─ win 1  claude  --worktree feat-x        (coder, Max)
           │     ├─ win 2  claude  --worktree feat-y        (second coder; talks to win 1 via SendMessage)
           │     ├─ win 3  codex                            (reviewer; or /codex:review from inside claude)
           │     ├─ win 4  devbox opencode                  (in the container; only this worktree mounted; cheap models)
           │     └─ win 5  claude remote-control            (phone access); `claude --bg` for long jobs
           │     ~/dev/<client>/...   authoritative code; worktrees under .claude/worktrees/; remotes on GitHub
           │     ~/dev/freelance-practice, ~/dev/infra-docs  context clones: --add-dir + sandbox denyWrite
           ├─ automation lane: auto     (stage 2; timers; own API keys; write key ONLY to freelance-automation-inbox repo)
           └─ service lane: n8n         (stage 2; own user + private podman network; no code/vault mounts)
```

Vault dataflow: the laptop's Obsidian copies stay canonical (Claude Desktop MCP writes there). `dev` clones are context: added with `--add-dir` **and** listed in `sandbox.filesystem.denyWrite`, so "read-only" is enforced, not assumed; they only pull. `auto` holds **no key to the vault at all**; it writes to a separate repo `freelance-automation-inbox` with its own deploy key, and the laptop pulls that repo and moves notes into the vault deliberately. A prompt injection through ingested web content can at worst vandalise the inbox repo. (GitHub rulesets on `main` were the alternative; on a private repo they need a paid plan, and a key that can't reach the repo is the stronger control anyway.) Code repos are the reverse — VPS canonical; pulling them on the laptop should not produce path-level conflicts under this convention.

Loopback is open to `dev` in stage 1 (`opencode serve`, if used). That is a stated hole in the lane model, not isolation: keep an inventory of localhost listeners in the netcup ADR (like ADR-0031 on the laptop). When n8n arrives (2.6) it does **not** get a bare loopback port: private podman network, UI bound to the tailscale address behind Caddy, and a `skuid dev` drop rule for its port, so coding agents cannot address the box that will hold integration credentials.

## 3. Staged plan

Hours are estimates for you plus Claude Code; each task has a stopping point.

### Stage 0 — prerequisites on the VPS — **complete 2026-09-11**

| # | Task | Done when | h |
|---|---|---|---|
| 0.1 | ~~ADR: rewrite ADR-0021's residual into the enforceable credential policy~~ **Done 2026-09-11** as `netcup-vps/decisions/20260911-persistent-credentials-on-netcup-vps.md` (ADR-0034, proposed; accept after stage 1 matches the table) | Record in `netcup-vps/decisions/`, hooks pass | 1 |
| 0.2 | ~~restic → B2~~ **Done 2026-09-11.** Daily timer, `LoadCredential=`, monthly check, staleness → ntfy, restore verified from the laptop, retention from the laptop. **Measured:** the write-only key can *hide* but not *destroy* — 30-day prior versions are the recovery window (see ADR-0034 table, runbook §4a). Object Lock + recovery drill → stage 3 | `restic snapshots` lists timer-taken snapshots; restore verified; alert pipe tested | done |
| 0.3 | ~~netcup panel firewall as outer ring~~ **Done 2026-09-11** — policy `tailnet-only`, verified off-tailnet and across reboot; see `netcup-vps/sessions/20260911-panel-firewall-outer-ring.md` | Public IPv4 shows only the ports you chose; tailnet SSH still works after reboot | 1 |

### Stage 1 — coding rig — **complete 2026-09-12** (two evenings + one day)

| # | Task | Done when | h |
|---|---|---|---|
| 1.1 | Create `dev` (and, empty for now, `auto`), `loginctl enable-linger dev`, laptop **public** key into `~dev/.ssh/authorized_keys`, tmux, zram, podman (rootless, for `dev`), `MemoryMax=5G` on `dev`'s slice | `ssh vps-dev -t tmux new -As main` works from the laptop; `podman info` as `dev` works; `systemd-cgtop` shows the slice | 1 | **Done 2026-09-11 (also: AllowUsers gained `dev`).**
| 1.2 | nftables: `ct state established,related accept` first, then `skuid dev`/`auto` drops with **counters** for RFC 1918, 169.254.0.0/16, 100.64.0.0/10 (lo and everything else allowed) | From `dev`: `nc -zvw3 api.anthropic.com 443` connects; `nc -zvw3 <TS_IPV4_arch-thinkpad> 22` fails **and** the drop counter incremented; an existing and a fresh inbound tailnet SSH session both survive activation | 1 | **Done 2026-09-11 — measured 0/1/0 and counters; container traffic also caught.**
| 1.3a | **Claude Code** as `dev`: install, `/login` paste-code flow, `chmod 700 ~/.claude`; `apt install bubblewrap socat`; user settings: `sandbox.enabled`, `failIfUnavailable: true`, `allowUnsandboxedCommands: false`, `sandbox.credentials` deny on `~/.ssh/**`, `~/.claude/.credentials.json`, `~/.codex/**`, `~/.config/secrets/**`, `filesystem.denyWrite` on the two vault clones, `permissions.deny` mirror + `blockReadsOutsideWorkingDirectories` (snippet §5) | `/sandbox` shows no Dependencies tab; sandboxed `cat ~/.ssh/config` and `touch ~/x` denied; `touch ~/dev/infra-docs/x` denied while `cat` there works; removing bubblewrap makes `claude` refuse to start | 1.5 | **Done 2026-09-11 — Claude Code 2.1.269; all three tests pass (denied writes fail silently, verify with ls).** **Done 2026-09-12.**
| 1.3b | **Codex** as `dev`: install, `codex login --device-auth`, `chmod 700 ~/.codex`; `config.toml` with `default_permissions = "reviewer"` and two `[permissions.*]` profiles — `reviewer` (read-only source, secrets denied, no network) and `builder` (workspace write, secrets denied); **remove** `sandbox_mode` (docs: don't combine) | `codex` in `reviewer` refuses to write; both profiles refuse `~/.ssh` and `~/.config/secrets`; note which profile `codex-plugin-cc` actually applies (unknown §4) | 1 | **Done 2026-09-11 — codex-cli 0.154.0; deny `~/.codex/auth.json` not `~/.codex` (binary lives there); `glob_scan_max_depth = 6`.**
| 1.3c | **OpenCode** in the container: adapt the devbox recipe (drop Claude/Codex layers, add `opencode`), wrapper `devbox opencode` mounting only `$PWD` (a worktree) and `~/.local/state/devbox/opencode`; `opencode auth login` inside once; OpenCode `permission` block as belt to the container's braces | From inside: `ls ~/.ssh` fails (not mounted); `cat /work/README.md` works; `nc -zvw3 <TS_IPV4_arch-thinkpad> 22` fails (skuid rule reaches pasta traffic) | 2 | **Done 2026-09-11 — Debian image, OpenCode 1.18.30, uid 1001; tailnet blocked from inside; logged in OpenAI + DeepSeek. Runbook: `netcup-vps/runbooks/coding-rig-stage-1.md`.**
| 1.5 | GitHub credentials: **one deploy key per repo** (`~dev/.ssh/gh-<repo>` + `Host gh-<repo>` alias in `~dev/.ssh/config`, `IdentitiesOnly yes`), script that generates key + alias + prints the public key; laptop private key never leaves the laptop. Fine-grained PAT (repo-scoped, expiring) as the escape hatch when one job spans repos. Close `vps-trust-direction`. **Pushes are human actions (D4a):** `!git push` from the Claude window or a plain tmux shell; sandboxed commands cannot read the key | `ssh -T gh-<repo>` succeeds for two repos with two keys from your shell; a sandboxed `git push` inside Claude fails on the key; none of the keys is in arch-thinkpad's `authorized_keys` | 1 |
| 1.6 | Workflow: `claude --worktree` per task; `claude --bg` for long jobs, `claude agents` to watch them; two sessions message each other | Two worktree sessions coordinate via `SendMessage` without you copy-pasting | 0.5 | **Done 2026-09-12 — messaging and Remote Control on by default.**
| 1.7 | Review gate: `codex-plugin-cc` in Claude Code; skills layout `.agents/skills/` + `ln -s ../.agents/skills .claude/skills`; `ralph-loop` plugin | `/codex:review` runs on a test diff; a skill in `.agents/skills` shows in `claude`, `codex` and `opencode` | 1 | **Done 2026-09-12 — four sandbox openings needed, recorded in ADR-0035 §3; first review found 2 real defects.**
| 1.8 | Phone path: `claude remote-control` in tmux window 5; Termux + Tailscale as fallback | Claude app on the phone drives a VPS session; `ssh` from Termux attaches to tmux | 1 | **Done (default-on).**
| 1.9 | Laptop: `~/.ssh/config` `Host vps-dev` (User dev, **no ForwardAgent**), alias `dev='ssh -t vps-dev tmux new -As main'`; keep the local devbox for offline work | One command lands you in the session | 0.5 | **Done.**
| 1.10 | Backup scope: include `~dev/dev`, `~dev/.claude/projects`, `settings.json`, `~/.codex/config.toml`, `~/.config/opencode`; exclude `.credentials.json`, `auth.json`, `node_modules`, `.cache` (mirror ADR-0017) | Restore of one transcript file from B2 verified | 1 | **Done — timer run verified.**
| 1.11 | Client project template repo: `CLAUDE.md` = `AGENTS.md`, `.agents/skills/`, worktree + review-gate convention, `deploy.sh` (human-run, sources `~/.config/secrets/<client>.env`, 0600, then `wrangler pages deploy`), Impressum/Datenschutz placeholders, Lighthouse/a11y check script | Template instantiated for the Chantal site; first `/codex:review` passed; `deploy.sh` run by you succeeds, run under the sandbox fails on the env file | 2 | **Done — `Luis-Gordon/site-template`, Astro 5 on Node 22, Lighthouse 12×100.**
| 1.12 | Document: ADR "netcup-vps is the coding host" (extends ADR-0033 to a second device; references 0.1), localhost-listener inventory, runbook "coding session start/stop/recover", update netcup `_index.md` and backlog | Records exist in `infra-docs/netcup-vps/`, hooks pass | 1 | **Done — ADR-0035 accepted; listener inventory deferred to 2.6.**

Stage 1 exit: you code the Chantal site from any device; a reviewer from a different model family gates every merge; no agent can push, deploy or read a credential; a laptop failure costs nothing.

### Stage 2 — automations (≈20 h) — **redesigned 2026-09-12**, see `infra-docs/_scratch/2026-09-12-stage-2-automations-design.md`

Shape changed: producers (timers as `auto`) → one inbox repo → a chat-first **secretary** (Hermes, contained as a fourth principal, no skills, tailnet-only gateway, own capped keys) as step 2.8. Hermes is acceptable *contained and last*; OpenClaw stays out. The table below is superseded by the design note's.

Order chosen so each step produces something you can later sell.

| # | Automation | Trigger → output | Tool | Failure behaviour | h |
|---|---|---|---|---|---|
| 2.1 | Alert channel | any unit failure → phone push | ntfy (self-host behind Tailscale, or ntfy.sh topic with access token); `OnFailure=` drop-in for every timer | none (this *is* the failure channel); healthchecks.io heartbeat as the watcher's watcher | 1 |
| 2.2 | Trend digest (AI coding, freelance, marketing) | daily 06:00 → Markdown committed to the `freelance-automation-inbox` repo (as `auto`, see 2.3) + ntfy summary | Miniflux + `miniflux-ai` or a 100-line Python script; DeepSeek/GPT-nano ≈ $5/mo at 300 items/day | retry once, then alert; stale marker like restic's | 3 |
| 2.3 | Admin triage | weekly → open to-dos from vault (`gruendung`, client timelines, `_inbox/`) ranked, dated, pushed | `claude -p` in a systemd timer as **`auto`** (own API key or setup-token; read-only vault clone pulled over HTTPS or via a *read-only* deploy key; **write key only to `freelance-automation-inbox`**), output committed and pushed there; the laptop pulls it and you move notes into the vault | alert on non-zero exit; `auto` physically cannot reach the vault repo's `main` — no key, no hook to bypass | 2 |
| 2.4 | Client research memo | per active client, weekly → "what changed in their niche/platforms" note | `claude -p` + WebSearch, template in `templates/` | same | 2 |
| 2.5 | Prospect research (compliant) | weekly → CSV of *businesses* (name, site, source URL, why) from directories, Impressum pages, job boards; 90-day deletion | Playwright/Crawl4AI script; no personal data beyond Impressum; outreach channel decided with the Mentor | alert; rate-limited, robots-respecting | 4 |
| 2.6 | Your own n8n | learning instance: Postgres + n8n compose, Caddy on the tailnet only, encryption key in 0600 env, in backup set | rootless podman compose **as its own `n8n` user** — no code or vault mounts, data exchange via webhooks/files it owns; ≈ 300–800 MB RAM | container restart policy; backup includes the DB dump | 3 |
| 2.7 | Hour log | tmux session start/stop → timesheet rows (Krankenkasse evidence per `business/model.md`) | tmux hooks + a one-line append; or `timew` | none needed | 1 |

Costs stage 2: ≈ $5–15/month API tokens; everything else runs on the existing VPS.

### Stage 3 — reconsider after 6–8 weeks of use

- A separate `CODEX_HOME` for the review plugin (closes ADR-0035 §3's opening); Astro 7 upgrade of the template with a review; `npm audit` triage.
- Devbox container for Claude/Codex too if the host sandboxes feel thin or you start handling client customer data. Conversely: OpenCode back onto the host (with an equivalent sandbox) if it gains peer-session messaging worth having across harnesses.
- Autonomous deploys via `sandbox.credentials` `mask` (token substituted only toward `api.cloudflare.com`).
- B2 Object Lock and a "recover repository after malicious hide" drill (measured 2026-09-11: the VPS key can hide snapshots; recovery = un-hide prior versions with the laptop's admin key within 30 days).
- Claude Squad / ccmanager if you routinely run more than two parallel sessions.
- OpenCode Go ($10/mo) if cheap-model usage via API exceeds that.
- Hermes Agent on a **separate** €5 VPS if you want a chat-first assistant; check the OAuth billing bug (#47260) and CVE cadence first.
- ChatGPT Pro upgrade based on two weeks of Codex usage numbers.
- Tailscale SSH — the existing backlog item; not changed by this plan.

## 4. Unknowns and the cheapest way to close each

| Unknown | Close it by |
|---|---|
| Per-session RAM of Claude Code / Codex / OpenCode on this box | `systemd-cgtop` during week 1; set `MemoryMax` from the numbers |
| Whether Plus's Codex quota survives review-per-merge | Count `/codex:review` calls for two weeks before upgrading |
| **Known, volatile:** as of 2026-09-11 `claude -p` draws from subscription usage; the announced Agent SDK credit is paused (Anthropic help-center article 15036540, updated June 16). Anthropic has said this may change | Re-read that article before stage 2.3/2.4; if a credit appears, meter it and define behaviour on exhaustion; until then timers stay small and Sonnet-sized |
| Which permission profile `codex-plugin-cc` applies when it drives Codex via app-server | Step 1.3b: run `/codex:review`, then check Codex's log for the profile name; if it ignores `default_permissions`, the review runs under the plugin's own sandbox settings — record which |
| `skuid` rules vs pasta: does OpenCode's container traffic leave as `dev` so the drop rules apply | Step 1.3c acceptance test (`nc` to the laptop from inside the container fails and the counter increments) |
| Which `permissions.deny` patterns Claude Code applies to Read/Glob (vs Bash) and whether `blockReadsOutsideWorkingDirectories` covers `~` | Step 1.3 acceptance test: ask each harness to read `~/.ssh/config`; the refusal is the check |
| `skuid`-keyed nftables and pasta (only matters when the container returns) | Stage 3 build-time test |
| Does Codex read `.claude/skills` today | `codex --help` / skills docs at install time; the symlink covers both directions |
| Exact Claude Code version pins for the VPS (laptop pins 2.1.233 / Codex 0.147.0) | `claude --version`, `codex --version` after install; record in the ADR |
| Termius Android mosh support | Irrelevant if Termux works; test once |

## 5. Commands and snippets for stage 1 (verify flags against `--help` first — infra-docs rule)

```bash
# 1.1 — as sudo user
sudo useradd -m -s /bin/bash dev
sudo loginctl enable-linger dev
sudo apt install tmux systemd-zram-generator
printf '[zram0]\nzram-size = ram / 2\ncompression-algorithm = zstd\n' | sudo tee /etc/systemd/zram-generator.conf
sudo systemctl daemon-reload && sudo systemctl start systemd-zram-setup@zram0.service
sudo systemctl set-property user-$(id -u dev).slice MemoryMax=5G
```

```nft
# 1.2 — /etc/nftables.d/agent-egress.nft (include from nftables.conf; keep Tailscale's tables)
table inet agent_egress {
  chain out {
    type filter hook output priority 0; policy accept;
    ct state established,related accept
    oifname "lo" accept
    meta skuid { "dev", "auto" } ip daddr { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16, 100.64.0.0/10 } counter drop
    meta skuid { "dev", "auto" } ip6 daddr { fc00::/7, fe80::/10 } counter drop
  }
}
```

Verification for 1.2, as `dev`: `nc -zvw3 api.anthropic.com 443` → succeeded; `nc -zvw3 <TS_IPV4_arch-thinkpad> 22` → fails; then as sudo user `nft list chain inet agent_egress out` shows the drop counter `packets` > 0. (`curl` to port 22 fails even when the TCP connect succeeds, so it proves nothing.)

```bash
# 1.3 — as dev
curl -fsSL https://claude.ai/install.sh | bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
curl -fsSL https://opencode.ai/install | bash
mkdir -p ~/.config/secrets && chmod 700 ~/.config/secrets
tmux new -As main
claude            # /login → copy URL to laptop → paste code
codex login --device-auth
opencode auth login
chmod 700 ~/.claude ~/.codex ~/.local/share/opencode
chmod 600 ~/.claude/.credentials.json ~/.codex/auth.json ~/.local/share/opencode/auth.json
```

```json
// 1.3a — ~dev/.claude/settings.json (user scope). Verify key names against
// https://code.claude.com/docs/en/settings-reference and /docs/en/sandboxing before use.
// sandbox.* is the enforcement (OS level, Bash + children); permissions.* is UI policy.
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "credentials": {
      "files": [
        { "path": "~/.ssh", "mode": "deny" },
        { "path": "~/.claude/.credentials.json", "mode": "deny" },
        { "path": "~/.codex", "mode": "deny" },
        { "path": "~/.config/secrets", "mode": "deny" }
      ]
    },
    "filesystem": {
      "denyWrite": ["~/dev/freelance-practice", "~/dev/infra-docs"]
    }
  },
  "permissions": {
    "additionalDirectories": ["~/dev/freelance-practice", "~/dev/infra-docs"],
    "blockReadsOutsideWorkingDirectories": true,
    "deny": [
      "Read(~/.ssh/**)", "Read(~/.claude/.credentials.json)", "Read(~/.codex/**)",
      "Read(~/.config/secrets/**)"
    ]
  }
}
```

```toml
# 1.3b — ~dev/.codex/config.toml. Verify table/key names against
# https://developers.openai.com/codex/config-reference at install time; do NOT keep sandbox_mode.
default_permissions = "reviewer"
approval_policy = "on-request"

[permissions.reviewer]
# read-only source access, tests only when asked, no secrets, no network
# (fill from the reference: filesystem read rules for the workspace, deny for ~/.ssh, ~/.config/secrets, ~/.codex/auth.json)

[permissions.builder]
# workspace write, same denies, network only where a task needs it
```

```bash
# 1.3c — OpenCode in the devbox container (wrapper sketch; adapt from ~/dev/devbox/devbox)
# mounts: the current worktree at /work, OpenCode state at /home/dev/.local/share/opencode. Nothing else.
podman run --rm -it --userns=keep-id --cap-drop=ALL --security-opt no-new-privileges \
  -v "$PWD":/work -v "$HOME/.local/state/devbox/opencode":/home/dev/.local/share/opencode \
  -w /work localhost/devbox-opencode:latest opencode "$@"
```

OpenCode's own `permission` block (narrow shell allowlist, per its docs) goes inside the container as a convenience layer; the container is the boundary. Acceptance test for all three: "print my ~/.ssh/config" must fail — refused by policy on the host harnesses, file absent in the container.

```bash
# 1.5 — per-repo deploy key, run as dev; repeat per repository
r=chantal-site
ssh-keygen -t ed25519 -N '' -C "netcup-dev-$r" -f ~/.ssh/gh-$r
printf 'Host gh-%s\n    HostName github.com\n    User git\n    IdentityFile ~/.ssh/gh-%s\n    IdentitiesOnly yes\n\n' "$r" "$r" >> ~/.ssh/config
cat ~/.ssh/gh-$r.pub     # → GitHub repo → Settings → Deploy keys → allow write
git clone gh-$r:<owner>/$r.git ~/dev/$r
```

```bash
# 1.7 — inside claude
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
# per task
claude --worktree feat-contact-form   # implement
/codex:review                         # independent GPT review of the diff
# skills shared by all three harnesses (run at repo root)
mkdir -p .agents/skills .claude && ln -s ../.agents/skills .claude/skills
```

```sshconfig
# 1.9 — laptop ~/.ssh/config (dotfiles repo, ADR-0030). The private key lives in
# KeePassXC's agent; naming the .pub with IdentitiesOnly is how ssh picks it
# (see devbox/reference/ssh-config.example). ForwardAgent stays off.
Host vps-dev
    HostName <TS_IPV4_netcup-vps>
    User dev
    IdentityFile ~/.ssh/id_ed25519_t14.pub
    IdentitiesOnly yes
    ForwardAgent no
```

## 6. What you had not listed

1. **Backups before anything else on the VPS** — the netcup backlog is explicit; daily coding there without them breaks your own rule.
2. **Spend caps** on every API key (Anthropic console, OpenAI, DeepSeek, OpenRouter) — the only real brake on a looping agent.
3. **A secrets convention for the VPS** — no KeePassXC there. 0600 env files under `~/.config/<tool>/env`, loaded via systemd `EnvironmentFile=`; document the file names, never the values, in the vault.
4. **A watcher for the watchers** — timers fail silently. `OnFailure=` → ntfy, plus one external heartbeat (healthchecks.io free tier).
5. **The client project template** (1.9) — this is the actual "prepare for coding" item; the rig is the prerequisite.
6. **Deploy credentials are client credentials** — a Cloudflare token on your VPS is a credential to *their* account: project-scoped, in the account inventory, rotated at handover (delivery standard).
7. **Cross-session messaging vs containers** — the one interaction most setups get wrong; decided in D4.
7a. **The Unix user is not the agent boundary** — once a harness runs as `dev`, what matters is which paths, sockets, hosts and git capabilities *that invocation* can exercise. Those are configured per harness (1.3), and the sandbox must fail closed (1.4).
8. **n8n licensing** — one instance per client, in the client's name; yours runs only your workflows.
9. **Outreach law** — decided in D8 before any scraper is written.
10. **Quota hygiene** — Max 5x is 15–35 Opus-hours/week: Sonnet default, Opus for planning and review, Codex for the second opinion, cheap models via OpenCode for boilerplate and tests.

## Sources (read 2026-09-10)

Claude Code: [legal & compliance](https://code.claude.com/docs/en/legal-and-compliance.md) · [authentication](https://code.claude.com/docs/en/authentication.md) · [plan limits](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan) · [cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging) · [agent teams](https://code.claude.com/docs/en/agent-teams) · [worktrees](https://code.claude.com/docs/en/worktrees.md) · [remote control](https://code.claude.com/docs/en/remote-control) · [sandboxing](https://code.claude.com/docs/en/sandboxing) · [LLM gateway](https://code.claude.com/docs/en/llm-gateway.md) · [routines](https://code.claude.com/docs/en/routines.md)
Codex: [repo/releases](https://github.com/openai/codex/releases/latest) · [pricing](https://learn.chatgpt.com/docs/pricing) · [sandboxing](https://learn.chatgpt.com/docs/sandboxing) · [auth](https://learn.chatgpt.com/docs/auth.md) · [subagents](https://developers.openai.com/codex/subagents) · [codex-plugin-cc](https://github.com/openai/codex-plugin-cc)
OpenCode: [providers](https://opencode.ai/docs/providers/) · [Go](https://opencode.ai/docs/go/) · [Zen](https://opencode.ai/docs/zen/) · [server](https://opencode.ai/docs/server/) · [skills](https://opencode.ai/docs/skills/)
Routers/models: [claude-code-router](https://github.com/musistudio/claude-code-router) · [LiteLLM](https://docs.litellm.ai/docs/tutorials/claude_responses_api) · [OpenRouter](https://openrouter.ai/docs/guides/guides/claude-code-integration) · [DeepSeek pricing](https://api-docs.deepseek.com/quick_start/pricing)
Orchestrators: [Claude Squad](https://github.com/smtg-ai/claude-squad) · [ccmanager](https://github.com/kbwo/ccmanager) · [Gas Town](https://github.com/steveyegge/gastown) · [Symphony](https://github.com/openai/symphony) · [Vibe Kanban shutdown](https://www.vibekanban.com/blog/shutdown) · [ralph-loop](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop)
VPS: [netcup RS G12](https://www.netcup.com/en/server/root-server) · [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh) · [Debian podman](https://tracker.debian.org/pkg/podman) · [zram-generator](https://github.com/systemd/zram-generator) · [restic B2](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html) · [Termux + Tailscale](https://www.skeptrune.com/posts/claude-code-on-mobile-termux-tailscale/)
Automation: [Hermes providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) · [Hermes CVEs](https://labs.cloudsecurityalliance.org/research/csa-research-note-hermes-agent-cves-20260504-csa-styled/) · [OpenClaw hardening](https://adversa.ai/blog/openclaw-security-101-vulnerabilities-hardening-2026/) · [Anthropic SDK billing pause](https://devops.com/anthropic-hits-pause-on-claude-agent-sdk-billing-change-for-now/) · [n8n license](https://github.com/n8n-io/n8n/blob/master/LICENSE.md) · [n8n SUL explained](https://nordflux.de/en/guides/the-n8n-sustainable-use-license-explained) · [ntfy comparison](https://www.bigiron.cc/guides/gotify-vs-ntfy-vs-apprise-vs-pushover) · [miniflux-ai](https://github.com/Qetesh/miniflux-ai)
Law (DE): [§7 UWG social DMs — Plutte](https://www.ra-plutte.de/zulaessigkeit-von-werbenachrichten-via-social-media/) · [IHK B2B outreach](https://www.ihk.de/nordwestfalen/recht/rechtsthemen/wettbewerbsrecht/werbung-per-telefon-telefax-oder-e-mail-3614212) · [Instagram scraping 2026](https://www.socialcrawl.dev/blog/instagram-scraping-2026)
