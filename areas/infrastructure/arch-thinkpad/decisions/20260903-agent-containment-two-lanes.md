---
type: decision
id: ADR-0033
aliases: [ADR-0033]
status: accepted
impact: high
provenance: inferred
device: arch-thinkpad
constrains: spec/devenv, spec/containers
stale: false
created: 2026-09-03T01:30:00Z
updated: 2026-09-03T02:00:00Z
last_verified: 2026-09-03
commit: pending
supersedes:
superseded_by:
tags: [decision, devbox, containers, agents, containment, egress, nftables, vscode]
---

# ADR-0033 — Agent containment: two lanes, tools in the image, block the inside

- **Status:** accepted 2026-09-03, with the operator's amendments to points 3–5 below; nothing here is built yet
- **Provenance:** inferred (design), on measured premises named below

## Summary

Agents run in one of two lanes. **Coding** happens in the devbox: a container whose
toolset is a property of the image, with reference material mounted read-only and
network egress that reaches the internet but not this machine, the LAN or the
tailnet. **Administration** happens on the host as root, governed by the vault,
snapshots and the runbooks, and is not a container problem. Agent permission
prompts are convenience inside a lane, never the boundary.

## Context

Measured 2026-09-02/03:

- The devbox mounts three things — the project directory and the two agent state
  directories — and nothing else. Its package list is the hand-picked line in the
  `Containerfile` ([[../devenv/_index|devenv/]]). Every tool an agent turned out
  to need has been added one at a time after the container refused it; the
  operator named this loop as the problem.
- The local ArchWiki (`arch-wiki-lite`, `arch-wiki-docs`) is installed on the host
  and absent from the image (`podman run … pacman -Q` reports neither), so the
  vault's lookup rule has no channel 3 where the coding happens
  ([[../backlog/devbox-wiki-mount|devbox-wiki-mount]]).
- Egress from the container is unrestricted. The host's nftables `forward` chain
  is `policy drop` but rootless podman with pasta does not traverse it; a process
  in the container can reach the tailnet, the LAN and the host's loopback.
- Every task in the 2026-09-02/03 sessions — sshd, restic units, AppArmor, LUKS,
  usbguard, the collector — needed root on the host and access to `/etc`,
  `/var`, the journal and the block devices. No mount set makes that a container
  job, and the operator ran the agent as root on the host for it, against the
  advice of a general-purpose assistant that had not seen the vault.
- The operator's research needs are open-ended — arbitrary websites, not
  predictable in advance — so an egress *allowlist* is not workable.

## Decision

1. **Two lanes.** Coding in the devbox. Administration on the host, as root, with
   the root `CLAUDE.md` rules as the control: orient in the vault, recovery path
   before risk, measure and verify, confirm risky changes. Do not try to make one
   container serve both.
2. **Tools are a property of the image.** A tool an agent needs goes into the
   `Containerfile` package line, once, and is rebuilt in. Until the next rebuild
   the agent does without or the task moves to the host lane. No per-session
   installs inside a container.
3. **Reference material is baked into the image.** `arch-wiki-lite` and
   `arch-wiki-docs` go on the package line. No additional mounts from `$HOME`;
   the vault, when it is the project, is the project directory and arrives that
   way.
4. **Block the LAN and the host, allow the internet and the tailnet.** From the
   container, deny RFC 1918, link-local and the host's own addresses; allow
   everything else, **including the tailnet** — the operator expects automation
   work against n8n on netcup-vps from inside the coding lane, and the tailnet is
   already the trusted network (ADR-0027 governs what the VPS may hold). Denying
   the host's addresses breaks nothing today: no devbox workflow talks to a
   service on this laptop. The one thing it would block is `ollama` on the host,
   which becomes a named exception if local inference from a container is ever
   wanted. Mechanism to be measured at build time — pasta's own address-mapping
   options or an nftables `output` rule keyed on the container's uid/cgroup — and
   recorded as a spec step.
5. **The devbox wrapper is the default entry; VS Code is made as safe as the
   wrapper rather than replaced.** The operator works in terminal workspaces now
   and prefers them on one screen, so the wrapper is the daily path. The
   devcontainer's one real exposure is that the extension forwards
   `SSH_AUTH_SOCK` unconditionally (ADR-0003). The fix is upstream of the
   extension: **launch VS Code without the agent socket in its environment** (a
   desktop entry or wrapper that unsets `SSH_AUTH_SOCK`), so there is nothing to
   forward; pushes from that project happen in a wrapper terminal with
   `--with-agent`. That makes the two paths equivalent without changing IDE. The
   `.bashrc` shim from `devcontainer-ipc-bridge-hardening` still goes into the
   `Containerfile` as one `RUN` line, as belt to that brace. Cursor or another
   IDE is not needed for this and is not decided here.

## Alternatives Considered

1. **One container for everything, with more mounts and sudo.** A container with
   `/etc`, `/var`, the journal, block devices and root is the host with extra
   steps and a false sense of a boundary. Rejected; the host lane is honest about
   what it is.
2. **Egress allowlist.** The textbook containment and incompatible with the
   operator's stated research pattern. Rejected on that fact, not on principle.
3. **Trust the agent's own permission prompts as the boundary.** They are
   configurable by the agent's own config files, which live in a mounted state
   directory the agent can write. Rejected as a boundary; kept as a convenience.

## Consequences

### Positive

- The tool loop ends: the package line is the list, and a rebuild is the act.
- A compromised or confused coding agent cannot reach netcup-vps, pixel-graphene
  or services on this laptop, while research keeps working.
- The host lane's risk is named and governed rather than denied.

### Negative

- A tool missing from the image is a rebuild away, not a `pacman -S` away. The
  snapshot-pinned image (ADR-0016) makes a rebuild deliberate by design.
- The host lane has no technical containment at all. Its controls are
  procedural: snapshots, the confirm-before-risk rule, the vault. Two of today's
  mistakes — a `pkill` pattern that matched the agent's own shell, a clipboard
  that expired into a LUKS keyslot — happened in that lane and were recoverable
  because of those controls, not because of a sandbox.
- The tailnet stays reachable from the coding lane, so a compromised agent there
  can reach netcup-vps and pixel-graphene. Accepted on the operator's use case;
  the mitigation is what ADR-0027 already says about what the VPS holds, and
  the per-service credentials an agent would still need.
- VS Code launched without the agent socket cannot push from its own terminal;
  that is the point, and the wrapper terminal is one keystroke away.

## Verification (to be written as spec steps)

```bash
# arch-thinkpad, inside the devbox after rebuild
pacman -Q arch-wiki-lite arch-wiki-docs          # both present
wiki-search systemd-boot | head -1               # answers
curl -sI https://archlinux.org | head -1         # 200
ssh -o ConnectTimeout=5 netcup-tailnet true; echo $?  # tailnet reachable: 0 (with --with-agent)
curl -s --max-time 3 http://192.168.1.1 ; echo $?     # LAN denied: non-zero
curl -s --max-time 3 http://<host-gateway>:11434 ; echo $?  # host denied: non-zero
```

## Revisit when

A task appears that needs both lanes at once, or pasta gains a documented
deny-list that makes the nftables half unnecessary.

## Related

- [[20260818-devcontainer-agent-exposure|ADR-0003]], [[20260825-devbox-base-is-arch|ADR-0015]], [[20260825-devbox-pinning-four-layers|ADR-0016]], [[20260825-harness-state-and-identity|ADR-0017]]
- [[../backlog/devbox-wiki-mount|devbox-wiki-mount]], [[../backlog/devbox-cleanup-next-rebuild|devbox-cleanup-next-rebuild]], [[../backlog/devcontainer-ipc-bridge-hardening|devcontainer-ipc-bridge-hardening]], [[../backlog/devbox-contents-audit|devbox-contents-audit]] — all fold into the rebuild this record shapes
