# Block C — Contained Development Environment

*Built and verified 2026-08-17/18 on `arch-thinkpad`. **Complete.** Supersedes the v1 proposal.*

*Closes C.1 item 2 (contained development environment), C.3 "Dev toolchain", C.3 "VS Code telemetry".*

This document records what was built and what was measured. Where reality diverged from the plan or from vendor documentation, the measurement wins and the original assumption is kept alongside it — the wrong assumptions are the useful part.

---

## 1. What exists now

| Component | Location | Notes |
|---|---|---|
| Image | `localhost/devbox:latest` | Built from `~/dev/devbox/Containerfile`, base pinned by digest |
| Recipe repo | `~/dev/devbox` → `Luis-Gordon/devbox` (private) | Containerfile, `codex-config.toml`, `devbox` wrapper |
| Wrapper | `~/dev/devbox/devbox`, symlinked `~/.local/bin/devbox` | Sealed environment, opt-in agent via `--with-agent` |
| Editor path | Dev Containers extension, MS build of VS Code | `.devcontainer/devcontainer.json` per project |
| Agent state | `~/.local/state/devbox/claude`, `~/.local/state/devbox/codex-state` | Bind-mounted, host-side, survives `--rm` |
| Projects | `~/dev/<project>` → `/work` | On `@home`, restic-covered |

Versions at build time — the image is the version pin, not the Containerfile:

| Item | Version |
|---|---|
| Base | `docker.io/library/debian:trixie-slim@sha256:38a76d01668772e381ad2826d876627c89e7133e2f8a0f5d567306798b0f2a16` |
| Claude Code | 2.1.233 |
| Codex CLI | 0.147.0 |
| podman | 6.0.2, rootless, crun, netavark + pasta |

Repo contents at close of session (`Luis-Gordon/devbox`, `590de3a`):

```
Containerfile                            # image recipe, base digest-pinned
codex-config.toml                        # seeded to host state on first run
devbox                                   # wrapper, symlinked into ~/.local/bin
devcontainer-template/devcontainer.json  # copy into a project as .devcontainer/
reference/ssh-config.example             # ~/.ssh/config with the §3.4 note
```

---

## 2. Design decisions and their rationale

| Decision | Choice | Why |
|---|---|---|
| Runtime | podman rootless | §6.3 already rejected Docker. **Verified 2026-08-17:** `inet filter` ruleset hash identical before and after container use (`4bc77135…`), so the default-drop `input` chain and `forward policy drop` in §5.1 are untouched. Rootless podman writes no nftables rules. |
| Where agents run | Inside the container | The threat is a dependency reading `~/.ssh`, the KDBX or `~/Sync`. An agent on the host with a containerised build step does not close that; the agent process itself reads the host filesystem. |
| Host toolchain | No `nodejs`, `npm`, `python-pip`, `pipx` on the host | Per C.3, anything under `~/.local/bin`, `/opt`, pipx or `npm -g` matches no AppArmor attachment path and is unconfined regardless of §4.3. Keeping the host free of language package managers makes the container the path of least resistance. |
| Agent state | Bind mounts from `~/.local/state/devbox` | Named volumes live in `~/.local/share/containers`, outside both snapper and restic (§6.3). Auth would vanish on any image prune. |
| Base image | Debian stable, digest-pinned | A rolling base defeats the point of a reproducible artifact. The host stays Arch; the box does not have to match. |
| SSH in `devbox` | Absent by default, `--with-agent` opts in | Deliberate. The wrapper is the sealed environment. |
| SSH in the attached devcontainer | **Forwarded, unavoidably. Accepted.** | See §6 — this cannot be disabled by configuration. Decision recorded rather than left as a gap. |

---

## 3. Measured findings

Everything here was believed otherwise before it was tested.

### 3.1 Codex's runtime lives in the directory you want to mount

`~/.local/bin/codex` is an absolute symlink into `~/.codex/packages/standalone/current/bin/codex`. Bind-mounting host state at `~/.codex` shadows the runtime and `codex` becomes `command not found`. The failure looks like a broken install, not a mount problem.

Fix: relocate **state** with `CODEX_HOME=/home/dev/.codex-state` and mount there. `codex doctor` confirms the split — runtime stays in the image layer and reports `install: consistent`, state and `config.toml` land in the mount.

Claude Code does not have this problem. Its binary lives at `~/.local/share/claude/versions/<ver>` and `~/.claude` holds only `backups/` and `downloads/`, both update machinery made moot by `DISABLE_AUTOUPDATER=1`.

### 3.2 Codex's inner sandbox works inside rootless podman

OpenAI's documentation warns that the bwrap/seccomp sandbox may fail to nest inside a container and recommends `danger-full-access` there. **On this machine that is false.** Measured under rootless podman with `--cap-drop=ALL`:

| Test | Result |
|---|---|
| `touch` inside the workspace, read-only mode | blocked |
| `touch` outside the workspace | blocked |
| `curl` to an external host | DNS resolution failed — blocked at the sandbox, not by network absence |

So `sandbox_mode = "workspace-write"` is used, not `danger-full-access`. This is a free second layer, and the network restriction on spawned commands is worth more than it looks: host `output` policy is `accept` (§5.1), and v1 §7 deferred egress control because the in-container firewall approach needs `NET_ADMIN`. Codex's sandbox achieves egress restriction for agent-issued commands with no capability grant at all.

**The container has full network.** Only commands Codex spawns are restricted. Codex's own API calls are unaffected — confirmed by `codex doctor` reaching `chatgpt.com/backend-api/` and receiving a real 401 from `api.openai.com`.

### 3.3 Codex auth needs `--device-auth`

The standard browser flow does not link back to the CLI from inside a container — the callback goes to the host's loopback, not the container's. `codex login --device-auth` sidesteps it entirely. **No port publishing is needed**, which retires the `-p 127.0.0.1:1455:1455` suggestion from v1.

Claude Code authenticates through its normal flow without incident.

### 3.4 `IdentitiesOnly yes` filters agent keys, not just on-disk keys

**This is the most broadly applicable finding and belongs in §9, not just here.**

§7.7 sets `IdentitiesOnly yes` under `Host *`. §7.7.1 noted the config still named a private key path that no longer exists and concluded *"Harmless — SSH tries the agent first."* That was an untested inference and it is wrong.

With `IdentitiesOnly yes`, SSH offers only identities named by an `IdentityFile`. With the private key held in KeePassXC and no matching `IdentityFile`, the agent's key is discarded before the offer. The trace is distinctive and reads like a server-side rejection:

```
debug1: get_agent_identities: agent returned 1 keys
debug1: No more authentication methods to try.
git@github.com: Permission denied (publickey).
```

The key is present, loaded, and never offered. There is no `Offering public key` line between those two.

**Fix:** point `IdentityFile` at the **public** key. SSH reads it for the fingerprint and requests that identity from the agent; the private half stays in the database.

```
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_t14.pub
    IdentitiesOnly yes
```

Applied to `github.com` (verified 2026-08-17) and to `hive`, `n8n`, `storagebox` (still untested — servers offline since July, but they carried the identical defect and would have failed the same way).

### 3.5 A locked KeePassXC database presents as an invalid key file

```
Load key "/home/luiswg/.ssh/id_ed25519_t14.pub": invalid format
git@github.com: Permission denied (publickey).
```

Nothing in that message points at KeePassXC. SSH is reporting that it could not load the `.pub` as a *private* key, which it only attempts after the agent supplies nothing. **First diagnostic for any sudden publickey failure is `ssh-add -l`**, not inspecting key files.

Same cause when the lock timer fires mid-session, and when the agent is cleared manually. Nothing is lost — lock and unlock re-adds the key.

### 3.6 `passt` was already installed

v1 predicted it missing and offered a `containers.conf` fallback. Both were unnecessary. `podman info` reports `netavark` as the network backend; rootless still routes through pasta.

### 3.7 An assembled bash array is not a used one

`git_args` was built and never expanded in the `podman run` invocation. `bash -n` passes — it is syntactically valid, merely dead. The only verification that catches this is a runtime check:

```bash
devbox bash -lc 'echo "$GIT_AUTHOR_NAME <$GIT_AUTHOR_EMAIL>"'
```

### 3.8 VS Code needs a Secret Service provider under Hyprland

No gnome-keyring, no KWallet, and Hyprland supplies nothing. VS Code offers "use weaker encryption," which is `--password-store=basic` — obfuscated with a hardcoded key, effectively plaintext, and readable by every unconfined process per §4.3. **Do not accept it.**

KeePassXC implements the freedesktop.org Secret Service API. Setup:

1. **Application Settings → Secret Service Integration** → enable
2. Create a dedicated group in the main window first (`Groups → New Group` → `SecretService`) — the settings dialog only *selects* existing groups, it cannot create one
3. Pencil icon in the *Manage* column → select that group

Everything in an exposed group is readable by any Secret Service client, so expose an empty dedicated group, never the group holding the SSH key attachment. Keep *Confirm when passwords are retrieved by clients* ticked — it is the per-request prompt that makes exposure tolerable.

Then force VS Code onto libsecret, since it does not always detect correctly under Hyprland:

```json
// ~/.vscode/argv.json
{ "password-store": "gnome-libsecret" }
```

Despite the name this does not require gnome-keyring — it means "use libsecret against whatever owns `org.freedesktop.secrets`."

Consequence: when the lock timer fires, VS Code loses the token and re-prompts. Consistent with the model. Also, the token now lives in the KDBX and therefore syncs to the Pixel and Proton Drive.

### 3.9 The Dev Containers extension forwards the SSH agent unconditionally

**Three configuration attempts, all ignored:**

| Attempt | Result |
|---|---|
| `remoteEnv: { "SSH_AUTH_SOCK": "" }` | Socket created anyway |
| `dev.containers.forwardSSHAgent: false` | **No such setting exists.** Invented during this session; do not look for it |
| `remoteEnv: { "SSH_AUTH_SOCK": "/dev/null/no-agent" }` | Overridden anyway |

This is a known upstream limitation with an open feature request (microsoft/vscode-remote-release #11413). The socket appears at `/tmp/vscode-ssh-auth-<uuid>.sock` regardless. The only documented workaround modifies VS Code's launch environment globally, not per project.

### 3.10 The attached devcontainer has IPC bridges back to the host

Measured inside the attached container:

```
SSH_AUTH_SOCK=/tmp/vscode-ssh-auth-<uuid>.sock
REMOTE_CONTAINERS_IPC=/tmp/vscode-remote-containers-ipc-<uuid>.sock
REMOTE_CONTAINERS_SOCKETS=["/tmp/vscode-ssh-auth-<uuid>.sock","/tmp/.X11-unix/X2"]
VSCODE_IPC_HOOK_CLI=/tmp/vscode-ipc-<uuid>.sock
VSCODE_GIT_IPC_HANDLE=/tmp/vscode-git-<hash>.sock
GIT_ASKPASS=…/extensions/git/dist/askpass.sh
BROWSER=…/bin/helpers/browser.sh
REMOTE_CONTAINERS_DISPLAY_SOCK=/tmp/.X11-unix/X2
```

`VSCODE_IPC_HOOK_CLI` and `REMOTE_CONTAINERS_IPC` are host command-execution bridges. `GIT_ASKPASS` and `VSCODE_GIT_IPC_HANDLE` are credential access via the host.

**This invalidates a premise of the v1 plan.** v1 §8.2 argued that attaching the editor was strictly a security improvement over a terminal. It is not. It is an improvement in one direction and a regression in another:

| Setup | Extension host | Container environment |
|---|---|---|
| Attached devcontainer | Contained ✓ | Host IPC bridges present ✗ |
| Unattached, `devbox` in integrated terminal | On host, executes project code ✗ | Clean ✓ |
| Attached **and** `devbox` in a separate terminal | Contained ✓ | Clean ✓ |

The third row is available at any time — two containers from the same image, both bind-mounting the project, edits visible in both. It costs the integrated-terminal convenience for agent sessions.

Note this is documented, intentional behaviour, not a Microsoft oversight. Dev containers were built for environment reproducibility; the bridges exist so `code .`, credential prompts, and `xdg-open` work from inside. The mismatch is using a reproducibility tool as a security boundary. Anthropic's own dev container documentation says the same thing — the container does not prevent a malicious project exfiltrating anything reachable inside it, and it is recommended only for trusted repositories.

---

### 3.11 restic was walking every snapper snapshot

Unrelated to Block C, found while applying the credential exclusion. The §8 invocation backs up `/home`, which includes `/home/.snapshots` — every snapper snapshot of `@home`, roughly 20 under the §8 retention policy.

Observed during a run: `/home/.snapshots/45/snapshot/luiswg/Videos/...`, i.e. restic descending into a snapshot and re-reading media it had already seen in the live tree.

**This is a scan-cost problem, not a storage problem.** Measured after the run:

| Metric | Value |
|---|---|
| Repo total (raw-data mode) | 7.617 GiB, 1.17× compression |
| Snapshot 434c1127 (Aug 12) | 542.928 MiB scanned |
| Snapshot e8761488 (Aug 18) | 73.005 GiB scanned |
| Aug 18 run | 189,689 files, 5:13, 7.455 GiB stored |
| Of those files | 186,844 "new", 163 changed, 2,682 unmodified |

Deduplication works correctly — 73 GiB scanned became 7.455 GiB stored. There is no accumulated waste and **no prune is needed**.

Two things make it worth fixing anyway:

**It is a step change, not a slow drift.** On 12 August the system was days old and snapper had barely accumulated anything, so the backup was 543 MiB. By 18 August the timeline had filled and the same backup scanned 73 GiB. Every further snapper tick adds another full copy of `~/Videos` to the scan set.

**The parent snapshot stops matching.** 186,844 of 189,689 files counted as "new" despite `using parent snapshot 434c1127`. Snapper's numbering means last run saw `/home/.snapshots/40/...` and this one `/home/.snapshots/45/...` — different paths, so restic treats byte-identical content as new files, re-hashes everything, and writes a full metadata tree that is stale by the next run.

Fix: `--exclude '/home/.snapshots'`. Snapshots are a local rollback mechanism on the same disk; they are not a backup and have no business in the off-site repo. Expect subsequent runs to return to roughly the 12 August profile.

Separately worth deciding: whether `~/Videos` belongs in restic at all. Downloaded media is bulk reproducible data — the same category as the container subvolume already excluded.

**Also surfaced:** two snapshots, six days apart, both manual. Per §8 that is by design since the T5 is not always attached, but a six-day gap spanning a new dev environment, SSH config changes and a new repo is more exposure than the design intends. Worth deciding whether "manual when the drive is connected" produces the cadence actually wanted.

### 3.12 The KeePassXC lock timer is now load-bearing

It fired twice during this session, both times mid-`git push`, and both times produced the §3.5 symptom. That is a friction cost, but the timer is also one of the compensating controls the §4 decision leans on: it bounds the window in which a local process — including anything in the attached devcontainer — can use the agent.

hyprlock at 900s idle (§7.5) already covers walk-up access, so the KeePassXC timer's distinct job is bounding *process* access, not physical access.

**Practice adopted:** unlock deliberately before pushing, rather than lengthening the timer. This keeps the key in the agent because it was put there for a specific task, which is closer to the `--with-agent` shape than an always-available agent would be. Record as a stated practice, not an accident of settings.

Alternative if the habit does not stick: KeePassXC's *"Lock databases when session is locked or the screen saver starts"*, paired with hyprlock, ties database state to presence rather than to a clock. Timer at 30–60 minutes as a backstop.

---

## 4. The decision on agent exposure

**Accepted: the attached devcontainer has the SSH agent and the host IPC bridges. This is a deliberate trade, not an unresolved gap.**

Reasoning:

- Removing it costs the integrated-terminal workflow, which is most of the reason for attaching the editor at all.
- The realistic threat is generic credential harvesting by a compromised dependency, which reads `$SSH_AUTH_SOCK`. A targeted attack that enumerates `/tmp` for this specific weakness is not the threat model for a personal machine.
- The remaining layers do real work: no host home, no `~/.ssh`, no KDBX, no `~/Sync` inside the container; Codex's inner sandbox restricting spawned commands; Claude's permission prompts and Codex's `approval_policy = "on-request"` putting a human in the loop before tool calls.
- Most developers using dev containers forward the agent without considering it. This setup is well above median even with the forward accepted.

**What this decision depends on.** The permission prompts are the control that catches an injected instruction. Relaxing Codex to `approval_policy = "never"` or running Claude with `--dangerously-skip-permissions` is where the containment story thins out — those should be a separate, deliberate decision, not drift.

**Note on prompt injection:** it does not require installing plugins or skills. Agents read repository content, READMEs, dependency manifests, pasted issue text, and fetched web pages; instructions embedded in any of those are the vector. "Don't install sketchy extensions" is not the mitigation — the approval prompts are.

**Compensating control, on the backlog:** diversify SSH keys. A GitHub-only key that may be forwarded freely, with the infrastructure key (`hive`, `n8n`, Storage Box) never leaving the host, changes a forwarded-agent compromise from "re-key everything" to "one forge account." This supersedes the one-key-per-device model in §7.7 and is worth doing on its own merits.

---

## 5. Configuration reference

### Containerfile

Current, as committed. The `~/.codex-default` COPY is vestigial — `CODEX_HOME` relocation made it unnecessary. Strike it at the next rebuild rather than rebuilding for cosmetics.

```dockerfile
FROM docker.io/library/debian:trixie-slim@sha256:38a76d01668772e381ad2826d876627c89e7133e2f8a0f5d567306798b0f2a16

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git openssh-client less jq ripgrep \
      build-essential python3 python3-venv python3-pip \
      nodejs npm \
 && rm -rf /var/lib/apt/lists/*

RUN useradd --uid 1000 --create-home --shell /bin/bash dev
USER dev
WORKDIR /home/dev
ENV PATH="/home/dev/.local/bin:${PATH}"

RUN curl -fsSL https://claude.ai/install.sh | bash
RUN curl -fsSL https://chatgpt.com/codex/install.sh | sh

COPY --chown=dev:dev codex-config.toml /home/dev/.codex-default/config.toml

ENV CLAUDE_CONFIG_DIR=/home/dev/.claude \
    CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 \
    DISABLE_AUTOUPDATER=1

WORKDIR /work
CMD ["/bin/bash"]
```

`CLAUDE_CONFIG_DIR` is **not optional**: Claude Code keeps its OAuth account and per-project trust in `~/.claude.json`, a file outside `~/.claude`. Mounting `~/.claude` alone leaves you signed out after every exit, and the symptom looks like a broken mount.

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` bundles telemetry, error reporting, bug command and auto-update opt-outs. It also disables feature-flag fetching, which switches off Remote Control and anything else depending on flags.

Both installers fetch latest at build time. Record `claude --version` and `codex --version` in the commit message on every rebuild.

### codex-config.toml

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
```

Seeded to `~/.local/state/devbox/codex-state/config.toml` by the wrapper on first use. The wrapper only copies when the target is absent, so **editing the repo copy requires deleting the seeded one** to take effect:

```bash
rm -f ~/.local/state/devbox/codex-state/config.toml
```

### Wrapper flags

| Flag | Effect |
|---|---|
| `--userns=keep-id` | Host UID 1000 ↔ container UID 1000. Files in `/work` are owned by `luiswg` on disk. Without it, files land as UID 100000+ and need `podman unshare` to clean up — the §9 trap. Also required for the `--with-agent` socket to be usable. |
| `--security-opt no-new-privileges` | No setuid escalation. |
| `--cap-drop=ALL` | Verified sufficient for image build, agent operation, and the VS Code server install. |
| cwd guard | Refuses to run outside `~/dev`, preventing a `$HOME` invocation from bind-mounting the entire home directory. The single most valuable line in the script. |
| `-e CODEX_HOME` | See §3.1. |
| `"${git_args[@]}"` | Identity passed as env, not by mounting `~/.gitconfig` — mounting drags in credential helpers, signing config and `includeIf` paths that reference host locations. |

### VS Code

Host settings at `~/.config/Code/User/settings.json`. **Written before VS Code was ever launched** — `~/.config/Code/` contained only `User/`, so no telemetry has been sent from this machine.

```json
{
  "telemetry.telemetryLevel": "off",
  "telemetry.feedback.enabled": false,
  "workbench.enableExperiments": false,
  "workbench.settings.enableNaturalLanguageSearch": false,
  "update.mode": "none",
  "update.showReleaseNotes": false,
  "extensions.autoCheckUpdates": false,
  "extensions.autoUpdate": false,
  "npm.fetchOnlinePackageInfo": false,
  "typescript.disableAutomaticTypeAcquisition": true,
  "git.autofetch": false,
  "security.workspace.trust.enabled": true,
  "security.workspace.trust.startupPrompt": "always",
  "dev.containers.dockerPath": "podman",
  "dev.containers.dockerComposePath": "podman-compose"
}
```

`update.mode: none` is right because `visual-studio-code-bin` updates through paru. `typescript.disableAutomaticTypeAcquisition` matters more than it looks — ATA fetches type packages from npm on the *host*, outside any container.

Host prerequisites for the extension:

```bash
systemctl --user enable --now podman.socket
loginctl enable-linger luiswg
```

Without linger the socket dies at logout and the extension fails to attach with an error pointing at Docker rather than systemd.

### devcontainer.json

```json
{
  "name": "devbox",
  "image": "localhost/devbox:latest",
  "remoteUser": "dev",
  "workspaceFolder": "/work",
  "workspaceMount": "source=${localWorkspaceFolder},target=/work,type=bind",
  "runArgs": ["--userns=keep-id", "--security-opt", "no-new-privileges", "--cap-drop=ALL"],
  "mounts": [
    "source=${localEnv:HOME}/.local/state/devbox/claude,target=/home/dev/.claude,type=bind",
    "source=${localEnv:HOME}/.local/state/devbox/codex-state,target=/home/dev/.codex-state,type=bind"
  ],
  "containerEnv": {
    "CODEX_HOME": "/home/dev/.codex-state",
    "CLAUDE_CONFIG_DIR": "/home/dev/.claude",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "DISABLE_AUTOUPDATER": "1"
  }
}
```

References the prebuilt image rather than building from the Containerfile, because a devcontainer build context cannot reach outside the project directory and copying the Containerfile into every repo would fork the recipe. One image, two entry points.

`CODEX_HOME` must be here too — the wrapper's env var does not apply, and without it the extension mounts over `~/.codex` and reproduces the §3.1 failure.

The `remoteEnv` block was removed; it does nothing (§3.9).

---

## 6. Uninvited software

The Copilot Chat extension installs a **Copilot CLI agent into the container** without asking:

```
/home/dev/.vscode-server/data/User/globalStorage/github.copilot-chat/copilotCli/copilot
```

More relevant than the vendor push: it is an agent with shell access that is not in the Containerfile, will not survive a rebuild, and is not in git. That is exactly the property the image-as-recipe model exists to prevent. Copilot Chat is preinstalled and unremovable in recent MS builds but can be disabled per-workspace or globally.

Worth periodically checking what else has arrived:

```bash
ls ~/.vscode-server/data/User/globalStorage/
```

---

## 7. What this does and does not protect

**Removed from reach of project dependencies and agent-issued commands:** `~/.ssh`, the KeePassXC database, `~/Sync`, Firefox profiles, `~/.config/containers/auth.json`, the entire host home directory. Verified by direct test.

**Not protected:**

- The bind-mounted project itself, including `.git`. A hostile dependency can install a `pre-push` hook that runs on the *host*. Check `core.hooksPath` and `.git/hooks/` before pushing — the one path where container-side writes execute host-side.
- Agent credentials. `~/.local/state/devbox/claude/.credentials.json` and `codex-state/auth.json` are live OAuth tokens in plain 0600 files, reachable from inside the container.
- The attached devcontainer's environment (§3.10, §4).
- Kernel-level isolation. A container is a process-isolation mechanism. For genuinely untrusted code the `qemu`/`libvirt` stack is the stronger boundary.

**New host-side exposure created by this block**, all readable by Obsidian and VS Code's UI process per §4.3:

- `~/.local/state/devbox/claude/.credentials.json`
- `~/.local/state/devbox/codex-state/auth.json`
- `~/.config/gh/hosts.yml` (GitHub CLI OAuth token — `gh` is host-only by design, since its token is a copyable file rather than an agent socket)

---

## 8. Remaining work

### Completed 2026-08-18

| # | Item | Outcome |
|---|---|---|
| 1 | restic exclusion for agent credentials | ✅ Applied and run. Tokens regenerate with one login; backing them up buys nothing and puts long-lived credentials in the repo. Snapper `home` still captures them — unavoidable without another nested subvolume, not worth one. Run also surfaced §3.11 |
| 2 | `.devcontainer/` into the devbox repo | ✅ As `devcontainer-template/`, not a live `.devcontainer/` — this repo is not itself opened in a container, and a live one would try to attach when editing the wrapper. `remoteEnv` stripped, since it does nothing (§3.9) |
| 3 | Commit the SSH config fix | ✅ `reference/ssh-config.example`, with the §3.4 note prepended |
| 4 | Verification checklist | Items 2–9 verified in session. **Item 12 (Extensions pane shows language extensions under "Container", not "Local") not yet confirmed by eye.** Item 1 full-ruleset hash is expected to drift — it includes Tailscale's iptables-nft tables; `inet filter` is the real test and matched |

### Still open

| Item | Note |
|---|---|
| Apply `--exclude '/home/.snapshots'` | §3.11. Worth ~20× the scan work on every backup |
| Drop this document into `~/dev/devbox/` | So recipe and reasoning version together |
| Apply §9 doc updates to the state reference | The §3.4 and §3.5 findings belong in that document's §9, not only here |
| Confirm checklist item 12 | One glance at the Extensions pane |

### Optional — security hardening if you want to raise the floor

**IPC bridge hardening.** A `.bashrc` shim in the image that unsets the host bridges (§3.10). Any shell in the attached container — and therefore Claude Code, Codex, and everything they spawn — sees no agent socket and no IPC handles.

```dockerfile
RUN mkdir -p /home/dev/.config && cat > /home/dev/.config/security-harden.sh <<'HARDEN'
unset VSCODE_IPC_HOOK_CLI
unset VSCODE_GIT_IPC_HANDLE GIT_ASKPASS VSCODE_GIT_ASKPASS_MAIN VSCODE_GIT_ASKPASS_NODE VSCODE_GIT_ASKPASS_EXTRA_ARGS
unset REMOTE_CONTAINERS_IPC REMOTE_CONTAINERS_SOCKETS REMOTE_CONTAINERS_DISPLAY_SOCK
unset WAYLAND_DISPLAY
export BROWSER=
export SSH_AUTH_SOCK=
export GPG_AGENT_INFO=
HARDEN
RUN sed -i '1i source ~/.config/security-harden.sh 2>/dev/null || true' /home/dev/.bashrc
```

**What it buys:** stops opportunistic use. The realistic threat — a compromised npm package doing generic credential harvesting — reads `$SSH_AUTH_SOCK`. This removes that.

**What it does not buy:** the sockets still exist as files. Anything deliberately enumerating `/tmp` finds them. Mitigation, not a boundary.

**Cost:** `git push` from the integrated terminal stops working. Push from a host terminal, or re-export the socket path deliberately per session — which is the `--with-agent` shape achieved differently.

Worth adding at the next rebuild regardless, since a rebuild is already needed to strike the vestigial `COPY`. It costs nothing when the agent is not in use.

### Backlog

| Item | Note |
|---|---|
| SSH key diversification | GitHub-only key forwardable; infrastructure key host-only. Supersedes the §7.7 one-key-per-device model. |
| KeePassXC per-key SSH confirmation | Trial for a fortnight. Converts silent use into a visible prompt. Decision criterion is prompt fatigue, not security — the security argument already favours it. |
| Exclude `~/Videos` from restic? | §3.11. Bulk reproducible data, same category as the container subvolume |
| Review the unlock-before-push habit | §3.12. If it does not stick within a fortnight, switch to session-lock integration rather than simply lengthening the timer |
| Backup cadence | §3.11. Six days between the only two restic snapshots. Decide whether manual-when-attached is the intended cadence |

**Resolved:** the `(1)` in the KDBX filename is a leftover from merging two devices' databases, not a sync conflict. `pixel-graphene` and `arch-thinkpad` act on the same file. No divergence.
| Disable Copilot Chat | §6. |
| Per-project image layers | The base is deliberately generic. Extend in a project-local Containerfile rather than bloating it. |
| `--name` collision in the wrapper | Two `devbox` invocations on the same project directory collide. Drop or suffix `--name` if it becomes annoying. |
| `nodocker` file | `sudo touch /etc/containers/nodocker` silences the `podman-docker` banner on stderr. Cosmetic, but some tooling parses stderr. |

---

## 9. Documentation updates required

| Document | Section | Change |
|---|---|---|
| State reference | §4.2 | Add `podman.socket` (user) and the `enable-linger` requirement |
| State reference | §4.3 | Revise "Residual after this work completes" — dependencies and agent commands no longer reach `~/.ssh`; Obsidian and the VS Code UI process still do. Narrowed, not closed. Add the three new host-side credential files from §7 |
| State reference | §6.3 | devbox layout; bind-mount-not-named-volume rule; digest pinning; **the measured nftables result with date** |
| State reference | §7.7 | `github.com` host entry added and verified 2026-08-17; the other three corrected but still untested |
| State reference | §7.7.1 | **Replace the "harmless" paragraph** with §3.4. Add §3.5 as the diagnostic note |
| State reference | §8 | Both restic excludes: `~/.local/state/devbox` (applied) and `/home/.snapshots` (§3.11, pending). Note the snapshot finding explicitly — it is the kind of thing that gets silently re-introduced |
| State reference | §7.7.1 | Add §3.12 — the unlock-before-push practice and why the lock timer is now load-bearing rather than a convenience setting |
| State reference | new §7.8 | devbox wrapper, its flags, the `--with-agent` opt-in, and the accepted devcontainer exposure from §4 |
| State reference | §9 | Add §3.4 and §3.5 — both are diagnostics that point somewhere other than the actual cause, which is what that section is for |
| State reference | C.1 item 2 | → Done 2026-08-18 |
| State reference | C.3 | Strike "VS Code telemetry" and "Dev toolchain" |
| State reference | C.4 | fcitx5 is **running** (PID observed, virtual keyboard attached) despite the user unit reading `inactive` — something else starts it. The "inert" note is stale |
| State reference | C.4 | Add `~/dev/devbox` to chezmoi scope. Note the two do different jobs: this repo versions the recipe, chezmoi versions the dotfiles that reference it. Do not nest them |

---

## 10. Rollback

Nothing here touches the boot chain, firewall, or encryption. No recovery path is at risk.

```bash
podman rm -af && podman rmi -af
podman system reset                  # nuclear
rm -rf ~/.local/state/devbox         # forces re-login
rm ~/.local/bin/devbox
systemctl --user disable --now podman.socket
```

VS Code settings roll back by removing the added keys. `~/.ssh/config` changes should be kept regardless — they fix a real defect independent of this block.

The one thing not trivially reversible: revoking agent OAuth sessions. Deleting local files does not invalidate tokens. If compromise is suspected, revoke server-side in the Claude and ChatGPT account settings.
