---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [security]
tags: [devcontainer, vscode, ipc, credentials]
created: 2026-08-21T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# IPC bridge hardening for the devcontainer

> [!NOTE] Summary
> The VS Code Dev Containers extension injects host IPC bridges into the container environment: `VSCODE_IPC_HOOK_CLI`, `REMOTE_CONTAINERS_IPC`, `GIT_ASKPASS`, and `SSH_AUTH_SOCK`. A `.bashrc` shim unsetting them raises the cost of opportunistic credential harvesting.

> [!INFO] Assessment 2026-09-03 — proportionality
> The operator asked whether this is necessary. Honest answer: it is cheap and it
> is not a boundary. What it defends against is code inside the container reading
> `SSH_AUTH_SOCK` and friends from the environment and using the host's agent
> without asking — an agent or an npm postinstall, not a human. ADR-0003 already
> accepts that the *devcontainer* path forwards the agent unconditionally; the
> `devbox` wrapper is the stronger path because it forwards only with
> `--with-agent` and only when the agent actually holds keys. So the real
> mitigation is **use the wrapper by default and the devcontainer only when VS
> Code's UI is worth it**, and this shim is a two-line hedge for the second case.
> Recommendation: fold the shim into the rebuild (it is one `RUN` line in the
> Containerfile) and do not spend a session on it. The broader question the
> operator raised — how to give agents a working toolset without re-adding tools
> one by one after locking the container down — is a design question, not this
> item; see the session log.

## Scope

**Mitigation, not a boundary.** It stops code that reads environment variables. It does not stop anything that enumerates `/tmp` for sockets. Do not record it as closing the gap.

## Steps

1. Add the unset shim to the container's `.bashrc` in the Containerfile.
2. Verify from inside an attached devcontainer that the variables are absent.
3. Confirm `git push` still works from a terminal inside the container, or accept that it does not and use the host.

## Related

- [[20260818-devcontainer-agent-exposure|ADR-0003]]
