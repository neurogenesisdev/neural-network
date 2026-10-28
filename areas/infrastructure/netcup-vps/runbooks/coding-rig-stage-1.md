---
title: "Coding rig stage 1: dev principal, egress rules, three harnesses"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-24
commit: pending
created: 2026-09-11
updated: 2026-09-24
tags: [runbook, devenv, agents, containment, nftables, podman, claude-code, codex, opencode, zsh, chromium]
---

# Coding rig stage 1 — dev principal, egress rules, three harnesses

> [!NOTE] Summary
> Stage 1 of `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md`
> on netcup-vps. **Done 2026-09-11:** 1.1 principals + tmux + zram + rootless
> podman + slice limits; 1.2 nftables egress keyed on uid; 1.3a Claude Code
> with fail-closed sandbox and credential denies; 1.3b Codex with `reviewer` /
> `builder` permission profiles; 1.3c OpenCode in a Debian container with only
> the worktree mounted, logged in to OpenAI (ChatGPT) and DeepSeek. **Done 2026-09-12:** 1.5 deploy keys, 1.6 worktrees + messaging,
> 1.7 codex-plugin-cc + skills + ralph-loop, 1.8 phone (Remote Control is on by
> default), 1.9 laptop alias, 1.11 `site-template` (Astro 5, Node 22), 1.12
> ADR-0035. **Stage 1 complete.** Everything below was run; what was
> *learned* is in the session log.
>
> **Amended 2026-09-23/24 — read §"Amendments" first:** Claude Code's Bash
> sandbox is **off** for `dev` and `blockReadsOutsideWorkingDirectories` is
> removed (§1.3a/§1.7 describe the old state); headless Chromium + browser MCP
> work; `dev`, `luiswg` and `root` run zsh.

Lanes: **admin** = `luiswg` + `sudo -i`. **coding** = `dev`. All three human
shells (`luiswg`, `dev`, `root`) are **zsh since 2026-09-24** — supersedes the
earlier "root stays bash"; see Amendments for the recovery-path check.
`auto` exists, empty, no SSH login.

## Paste rule (cost four attempts today)

A line that starts a new shell — `sudo -i`, `HISTFILE= zsh`, `ssh host` —
goes in a block **of its own**. Pasted at the top of a block it swallows every
line after it and nothing runs. And more than one heredoc per paste through
tmux gets garbled; one file per paste, `wc -l` after each.

## 1.1 — principals, terminal, resources (done)

```bash
# root
apt install -y tmux systemd-zram-generator podman uidmap passt slirp4netns netcat-openbsd
useradd -m -s /bin/bash dev; useradd -m -s /bin/bash auto
loginctl enable-linger dev
grep -E '^(dev|auto):' /etc/subuid /etc/subgid      # measured: dev 165536, auto 231072, 65536 each
install -d -m 700 -o dev -g dev /home/dev/.ssh
read -rp 'laptop public key: ' K; printf '%s\n' "$K" > /home/dev/.ssh/authorized_keys; unset K   # bash read
chown dev:dev /home/dev/.ssh/authorized_keys; chmod 600 /home/dev/.ssh/authorized_keys
printf '[zram0]\nzram-size = ram / 2\ncompression-algorithm = zstd\n' > /etc/systemd/zram-generator.conf
systemctl daemon-reload && systemctl start systemd-zram-setup@zram0.service   # measured: /dev/zram0 3.9G
u=$(id -u dev); mkdir -p /etc/systemd/system/user-$u.slice.d
printf '[Slice]\nMemoryMax=5G\nMemoryHigh=4G\n' > /etc/systemd/system/user-$u.slice.d/limits.conf
systemctl daemon-reload
# sshd: the hardening drop-in had AllowUsers luiswg — dev must be added, auto must not
sed -i 's/^AllowUsers luiswg$/AllowUsers luiswg dev/' /etc/ssh/sshd_config.d/10-hardening.conf
sshd -t && systemctl reload ssh
```

`~dev/.tmux.conf`: prefix `C-a`, mouse on, 50k history, base-index 1,
`tmux-256color`. Entry from the laptop: `ssh dev@<TS_IPV4> -t 'tmux new -As main'`.
Measured inside: `podman info --format '{{.Host.Security.Rootless}}'` → `true`;
slice shows `high: 4G, max: 5G`.

## 1.2 — egress rules keyed on uid (done)

Added to the existing `chain output` in `/etc/nftables.conf` (one table
`inet filter`; Tailscale owns its own tables — never `flush ruleset`):

```nft
ct state established,related accept
oif lo accept
meta skuid { "dev", "auto" } ip daddr { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16, 100.64.0.0/10 } counter drop
meta skuid { "dev", "auto" } ip6 daddr { fc00::/7, fe80::/10 } counter drop
```

`nft -c -f /etc/nftables.conf && nft -f /etc/nftables.conf`. Names resolve to
uids 1001/1002 at load. Measured as `dev`: `nc -zvw3 api.anthropic.com 443` →
0; `nc -zvw3 <TS_IPV4_arch-thinkpad> 22` → timeout; `127.0.0.1:22` → 0;
counter `packets 3` after the one blocked attempt (SYN retries).
`nftables.service` enabled. **Also measured: traffic from a rootless podman
container run by `dev` is caught by the same rule** (1.3c below) — the pasta
path leaves as the user's uid.

## 1.3a — Claude Code on the host (done)

Root: `apt install -y bubblewrap socat ripgrep git`. As `dev`:
`curl -fsSL https://claude.ai/install.sh | bash`; PATH line in `.bashrc`;
`claude --version` → **2.1.269**. `~/.claude/settings.json` (user scope):

```json
{
  "sandbox": {
    "enabled": true, "failIfUnavailable": true, "allowUnsandboxedCommands": false,
    "credentials": { "files": [
      { "path": "~/.ssh", "mode": "deny" },
      { "path": "~/.claude/.credentials.json", "mode": "deny" },
      { "path": "~/.codex", "mode": "deny" },
      { "path": "~/.config/secrets", "mode": "deny" },
      { "path": "~/.local/state/devbox", "mode": "deny" } ] },
    "filesystem": { "denyWrite": ["~/dev/freelance-practice", "~/dev/infra-docs"] }
  },
  "permissions": {
    "additionalDirectories": ["~/dev/freelance-practice", "~/dev/infra-docs"],
    "blockReadsOutsideWorkingDirectories": true,
    "deny": [ "Read(~/.ssh/**)", "Read(~/.claude/.credentials.json)", "Read(~/.codex/**)",
              "Read(~/.config/secrets/**)", "Read(~/.local/state/devbox/**)" ]
  }
}
```

Login: `/login` in the tmux session, URL opened on the laptop, code pasted
back. `/sandbox` shows no Dependencies tab. Measured: `cat ~/.ssh/authorized_keys`
→ "No such file" (denied paths read as absent — Claude itself flagged this and
that the override is off); `touch ~/x` → exit 0 **but no file anywhere**
(denied writes fail silently — verify with `ls` from a plain shell);
`curl https://api.anthropic.com` → reachable (the proxy's `200 Connection
Established` line is the proxy, not the server).

Note: `~/.codex` is denied here while Codex's own binary lives in
`~/.codex/packages/`. Fine for Claude's *sandboxed Bash*; if `codex-plugin-cc`
(1.7) fails to exec Codex from inside Claude, narrow this to
`~/.codex/auth.json` like the Codex profile below.

## 1.3b — Codex on the host (done)

`curl -fsSL https://chatgpt.com/codex/install.sh | sh` → **codex-cli 0.154.0**.
`codex login --device-auth`. `~/.codex/config.toml`:

```toml
approval_policy = "on-request"
default_permissions = "reviewer"

[permissions.reviewer]
description = "Read-only review of the workspace; no secrets, no network"
extends = ":read-only"
[permissions.reviewer.filesystem]
"~/.ssh" = "deny"
"~/.config/secrets" = "deny"
"~/.codex/auth.json" = "deny"     # NOT ~/.codex — the binary lives there; denying it breaks bwrap exec
"~/.claude" = "deny"
"~/.local/state/devbox" = "deny"
[permissions.reviewer.network]
enabled = false

[permissions.builder]
description = "Workspace writes; secrets denied"
extends = ":workspace"
[permissions.builder.filesystem]
glob_scan_max_depth = 6           # Linux bwrap has no native `**`; without this the .env deny is shallow
"~/.ssh" = "deny"
"~/.config/secrets" = "deny"
"~/.codex/auth.json" = "deny"
"~/.claude" = "deny"
"~/.local/state/devbox" = "deny"
[permissions.builder.filesystem.":workspace_roots"]
"." = "write"
"**/*.env" = "deny"
[permissions.builder.network]
enabled = true
```

Profiles are **beta** ("may change") — re-check the permissions page on
upgrade. Switch per session: `codex -c default_permissions=builder`.
Measured: reviewer refuses `~/.ssh` read, and a write *approved by the human*
still fails with "Read-only file system" — the kernel says no, not the model.
Builder writes inside the workspace. Whether `codex-plugin-cc` honours
`default_permissions` is still open (1.7).

## 1.3c — OpenCode in a container (done)

`~dev/devbox-opencode/`: `Containerfile` (debian:trixie-slim; node, npm,
python3, git, ripgrep, build-essential; user `dev` uid **1001** to match the
host; OpenCode via install script pinned to `OPENCODE_VERSION`), `build.sh`,
wrapper `opencode-box` (symlinked into `~/.local/bin`). Built with
`OPENCODE_VERSION=1.18.30`, image digest
`sha256:e58284c2f7fa3b04c885828f9b766bd1401e59bdac4e2be295b85fb36f1e3e0d`.

Wrapper mounts **two things**: `$PWD` (must be under `~/dev`) at `/work`, and
`~/.local/state/devbox/opencode` at `/home/dev/.local/share/opencode`
(`XDG_DATA_HOME` set so auth lands there). `--userns=keep-id --cap-drop=ALL
--security-opt no-new-privileges`. Measured from inside: `~/.ssh` absent,
`/work` is the project, tailnet **blocked**, internet OK. Logged in:
OpenAI via ChatGPT account, DeepSeek via API key. Anthropic key not added
(D2: Claude models run in Claude Code on the Max plan).

Rebuild = `OPENCODE_VERSION=<new> ./build.sh`; a `.env` or key never enters
the image.

## 1.7 — the review plugin's four openings (done, measured)

`codex-plugin-cc` runs Codex as a child of Claude's *sandboxed* shell. Four
things had to give, each found by running it — see the 2026-09-12 session log.
Net config in `~dev/.claude/settings.json` on top of §1.3a:

```json
"filesystem": {
  "allowRead":  ["~/.codex", "~/.local/bin"],
  "allowWrite": ["~/.codex", "~/.local/state/codex-plugin"],
  "denyWrite":  ["~/dev/freelance-practice", "~/dev/infra-docs"]
}
```

plus the `~/.codex` entries removed from `credentials.files` and
`permissions.deny`, Node from apt, and
`ln -s ~/.local/state/codex-plugin ~/.claude/plugins/data/codex-openai-codex`
(Claude Code refuses sandboxed writes under `~/.claude/plugins`; a symlink out
is followed). The `.claude/skills` symlink in a repo is created by a human —
the sandbox masks `.claude/` in the working directory. Trade-off recorded in
ADR-0035 §3. A hung review: `Ctrl+B Ctrl+B` backgrounds it.

Per task: `claude --worktree <name> --name <name>` → implement → commit on the
branch → `/codex:review --base master` → fix → review again → **you**
`git merge --ff-only` and `git push` (plain shell or `!` prefix). Every repo
starts with `git commit --allow-empty -m "empty root"` so a whole-tree review
has a base.

## Amendments 2026-09-23/24 (neurogenesis.dev redesign sessions)

### Sandbox off (Luis's decision, 2026-09-24) — `measured`

Over two days the Bash sandbox broke headless Chromium (`process_singleton_posix.cc:
socket() failed: Operation not permitted`), both browser MCP plugins, reads of
`/srv/vaults/*` (built-in deny list; `sandbox.filesystem.allowRead` was ignored in
2.1.278), `/tmp` screenshots and the plugin cache. Each fix needed another
exception. Decision: **`/sandbox` → Mode → "No Sandbox"** for `dev`, and the
`permissions.blockReadsOutsideWorkingDirectories` key deleted from
`~dev/.claude/settings.json` (it was the cause of the "read block" prompts, not
the sandbox). Measured after the change: Playwright MCP navigates and screenshots;
`scripts/shot.sh` works; reads of the vault paths no longer prompt.

Fences still in place (this is what the sandbox used to duplicate):
- `dev` is unprivileged, no sudo; per-repo deploy keys (ADR-0034).
- nftables `meta skuid` drops for `dev`/`auto` to private + tailnet ranges (§1.2) — unchanged, kernel-level.
- `permissions.deny` still blocks `Read(~/.ssh/**)`, `Read(~/.claude/.credentials.json)`, `Read(~/.config/secrets/**)`, `Read(~/.local/state/devbox/**)` (measured printout 2026-09-24).
- Branch per task, `/codex:review`, human `merge --ff-only` and push.

Trade-off accepted: the agent can read anything `dev` can read outside the deny
list and run any binary without a prompt in auto mode. `additionalDirectories`
(user scope) = `~/dev/freelance-practice`, `~/dev/infra-docs`.

**Measured 2026-09-24 16:47:** after choosing "No Sandbox" in `/sandbox`,
`~dev/.claude/settings.json` still had `"sandbox": {"enabled": true, "failIfUnavailable": true, ...}`
— the menu choice had been stored **per project** in
`~/dev/neurogenesis-dev/.claude/settings.local.json` (`"enabled": false`), so any
session outside that repo still had the sandbox. Fixed 2026-09-24 16:49: user-scope
`sandbox.enabled = false`; measured `/sandbox` shows "No Sandbox" in `~` and in the
repo. Lesson: `/sandbox` writes to the project's local settings, not user scope.
The §1.7 filesystem openings for codex-plugin-cc are now moot; `/codex:review`
ran fine without the sandbox on 2026-09-24.

### Headless Chromium for the browser tools — `measured`

- `/opt/google/chrome/chrome` → symlink to `/usr/bin/chromium` (root:
  `mkdir -p /opt/google/chrome && ln -s /usr/bin/chromium /opt/google/chrome/chrome`).
  Playwright MCP and chrome-devtools-mcp both probe only that path. Chosen over
  per-plugin `--executable-path` edits in the plugin cache, which a plugin update
  overwrites.
- `CHROMIUM_FLAGS="--headless=new --no-sandbox --disable-dev-shm-usage"` in
  `~dev/.zshenv`; Debian's `/usr/bin/chromium` wrapper appends it, so every
  launcher gets headless without per-tool config.
- Plugins (user scope): `frontend-design`, `playwright`, `chrome-devtools-mcp`
  (all `@claude-plugins-official`). Playwright measured working (390×844 shot,
  file under `.playwright-mcp/` or `.shots/`). chrome-devtools-mcp fails with
  "Missing X server" (headful by default) — still failing after `CHROMIUM_FLAGS`
  and after an attempted plugin edit (measured 2026-09-24): its launcher bypasses
  the Debian wrapper. **Uninstalled**; Playwright is the browser tool.
- Site repo helper `scripts/shot.sh <path|url> <WxH> [out]` (serves `dist/` on
  :4173, `pkill`s any `http.server 4173` when done). See the site runbook.

### zsh for the human users — `measured`

Root, once: `apt install -y zsh-autosuggestions zsh-syntax-highlighting fzf zoxide bat eza`
+ starship via its install script. Shared interactive config in
**`/etc/zsh/zshrc.local`** (history in `$XDG_STATE_HOME/zsh/history`, completion,
emacs keys, eza/bat aliases guarded by `command -v`, autosuggestions, fzf
bindings from `/usr/share/doc/fzf/examples/`, zoxide, starship, syntax
highlighting last). Per user (`dev`, `luiswg`, `root`):

- `~/.zshenv` — `PATH="$HOME/.local/bin:$PATH"` (+ `CHROMIUM_FLAGS` for `dev`)
- `~/.zshrc` — `source /etc/zsh/zshrc.local` + optional `~/.zshrc.local`
- `chsh -s /usr/bin/zsh <user>`; measured `/etc/passwd`: all three `/usr/bin/zsh`.

Service principals (`auto`, `n8n`, `hermes`, `mcpread`) keep their shells: their
units name an interpreter, and a zsh startup file could only add failure modes.
Rule: **humans get zsh, daemons get what their unit specifies.**

tmux: a running server keeps the shell it started with. `~dev/.tmux.conf` +=
`set -g default-shell /usr/bin/zsh`; old windows stay bash until closed or
`exec zsh`. `dev`'s tmux prefix is **`C-a`** (§1.1) — keep the laptop's tmux
on the default `C-b` or nested sessions fight over the key.

Recovery check (`inferred`): root is locked and reached only via `luiswg` +
`sudo -i` (runbooks/_index). A broken `/etc/zsh/zshrc.local` still leaves a
working zsh prompt (every source/eval is guarded). Worst case from a console:
`sudo -i bash` or `chsh -s /bin/bash root`. Not rehearsed.

### Laptop access (arch-thinkpad side) — `measured`

- `~/.ssh/config` gains a user-less host that lands in tmux (versioned per ADR-0030):
  ```
  Host netcup-vps          # Luis renamed it from `vps`; may shorten again
      HostName <TS_IPV4>
      IdentityFile ~/.ssh/id_ed25519_t14.pub   # public half; private key served by KeePassXC's agent
      IdentitiesOnly yes
      ForwardAgent no
      RequestTTY yes
      RemoteCommand tmux new -As main
  ```
  `ssh dev@netcup-vps`, `ssh luiswg@netcup-vps`. `RemoteCommand` breaks scp/rsync — use `vps-dev` for copies.
  tmux prefixes: laptop `C-b`, VPS `dev` `C-a` (checked 2026-09-24) — no clash when nested.
- Modern scp (SFTP mode) does not expand `{a,b}` remotely: `scp -r vps-dev:dev/neurogenesis-dev/.shots ~/Downloads/shots`.
- kitty: `shell tmux new -As local` in `~/.config/kitty/kitty.conf` (every window attaches the same session). Escape hatch: `kitty --override shell=zsh`.
- `vault-push` now pushes freelance-practice **and** infra-docs (source: `../build/auto/laptop-vault-push.sh`).

### Model alias — `measured`

`claude --model opus -p 'Reply with only your exact model id.'` → `claude-opus-5-5`
(2026-09-23): the `reviewer` subagent runs Opus 5.5 with no config change. See
[[model-routing]].

## Open steps (numbering from the plan)

All done 2026-09-12 except the localhost-listener inventory, which waits for the
first service (stage 2.6). Stage 3 items opened by this stage: a separate
`CODEX_HOME` for the review plugin; Astro 7 upgrade of the template with a
review; `npm audit` triage of the Lighthouse dependency tree.

## Related

- [[../decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — credential table; rows for Claude/Codex/OpenCode now exist on the host
- [[../sessions/20260911-panel-firewall-outer-ring|session log 2026-09-11]]
- `arch-thinkpad/_scratch/2026-09-10-ai-coding-rig-and-automation-plan.md` — the plan
