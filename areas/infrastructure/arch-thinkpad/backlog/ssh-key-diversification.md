---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [security]
tags: [ssh, credentials, keys, agent]
created: 2026-08-21T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# SSH key diversification

> [!NOTE] Summary
> One key (`id_ed25519_t14`) authorises GitHub, hive, n8n and the Storage Box. Because the VS Code Dev Containers extension forwards the agent unconditionally, a compromised dependency can sign with all four. Splitting the key changes the blast radius from "re-key everything" to "one forge account."

> [!INFO] GitHub split off 2026-09-03 — `measured`; the rest of the split is open
> New key `~/.ssh/id_ed25519_github` (ed25519, comment `github-only 2026-09-03`),
> public half added to the GitHub account, `Host github.com` in `~/.ssh/config`
> now names its `.pub` (committed in the dotfiles repo). Private key to be moved
> into KeePassXC and deleted from disk, per the existing model; until the agent
> holds it, GitHub pushes fail with *agent has no identities*. **The GitHub
> account still lists keys for `xps-windows` and `hive-vps`, both retired
> machines, and `vast-vm`** — three authorised keys whose private halves are on
> hardware the operator no longer controls. Deleting them is the higher-value
> half of this item and needs the operator's say. The old `id_ed25519_t14` key
> stays for the tailnet hosts; the container-forwarding half is
> `devcontainer-ipc-bridge-hardening`.

> [!SUCCESS] Closed 2026-09-03 — good enough
> GitHub has its own key; the retired machines' keys are deleted from the
> account; the private key lives only in KeePassXC. The tailnet hosts still
> share `id_ed25519_t14`, accepted: one key per device is available any time and
> nothing currently argues for it. The container-forwarding half is
> `devcontainer-ipc-bridge-hardening`.

## Proposal

- A **GitHub-only key** that may be forwarded freely into containers.
- An **infrastructure key** (hive, n8n, Storage Box) that never leaves the host and is never loaded while a devcontainer is attached.

Supersedes the one-key-per-device model currently documented for the fleet.

## Steps

1. Generate the second key; store in KeePassXC.
2. Add the infra key to `authorized_keys` on hive and n8n, and to Hetzner Robot for the Storage Box.
3. Update `~/.ssh/config` — remember `IdentityFile` must name the **public** key, see [[state-notes]].
4. Update `reference/ssh-config.example` in the devbox repo.
5. Remove the old key from GitHub once the split is verified.

## Blocked on

hive and n8n have been offline since July. Their `~/.ssh/config` entries were corrected on 2026-08-17 but remain **untested**.

## Related

- [[20260818-devcontainer-agent-exposure|ADR-0003]] — the exposure this mitigates
