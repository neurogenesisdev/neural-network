---
type: backlog-item
status: resolved
priority: medium
device: netcup-vps
projects: [netcup-build]
tags: [backlog, security, ssh, credentials, trust-model, github]
created: 2026-08-30
updated: 2026-08-31
commit: pending
---

# vps-trust-direction

> [!NOTE] Summary
> The operator asked for a server-side SSH key covering file copy to other
> devices, git authentication, and unspecified future use. The first of those
> inverts the direction trust currently flows in the estate.
> **ADR-0027 was accepted 2026-08-31**, so the principle is settled and the
> Tailscale ACL is unblocked. What remains is implementation: register the deploy
> key, and move the record to `shared/` with its inbound links updated.

## Context

Estate trust currently flows inward: arch-thinkpad and pixel-graphene hold
private keys, servers hold `authorized_keys`. This was never chosen — it fell out
of the per-device key model — which is exactly why it can be inverted without
anyone noticing.

netcup-vps is the worst candidate in the estate to hold a key that reaches
arch-thinkpad: it is the only machine with a public IP, the only one running
network services, and the one that will go weeks unattended. And the key would be
passphrase-free, because
[[../decisions/20260830-no-luks-on-always-online-vps|ADR-0021]] establishes there
is nobody present to type one.

The full argument and three alternatives are in
[[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]], **accepted
2026-08-31** — this Context section describes the situation as it stood when the
item was filed.

## Steps

1. ~~Decide ADR-0027~~ — **accepted as proposed, 2026-08-31.** The blocker was a
   misreading rather than a trade-off: the git case never required an upward
   credential, because GitHub sits *below* netcup-vps in the trust order. Only
   server-initiated `scp` was actually surrendered.
2. Implement the three replacements for the stated needs:
   - **File copy** — run `scp`/`rsync` from arch-thinkpad in both directions. No
     new credential. Add a note to the operator's own reference so the habit is
     available when wanted.
   - **Git** — a read-only deploy key per repository, generated on the VPS,
     public half added in that repository's settings. **Partially done**: an
     ed25519 keypair exists on the VPS and its public half is in KeePassXC
     (`measured` 2026-08-31). Registration in the repository's Deploy keys
     settings, the `Host` alias in the VPS's `~/.ssh/config`, and the
     `ssh -T` check are **not verified**.
   - **Future access** — Tailscale SSH with an ACL expressing
     `arch-thinkpad → netcup-vps` and denying the reverse. See
     [[tailscale-ssh-decision]].
3. ~~If rejected, the mitigated form~~ — moot; the record was accepted unamended.
4. Move ADR-0027 to `shared/` — it constrains every device, not this one. **Not
   done at acceptance**: `shared/` is empty, and five documents link to the
   record at its current path (both backlog items, the decisions index, the
   decisions `CLAUDE.md`, and the 2026-08-30 session log). The move and the link
   updates belong in one commit.

## Blocked on

Nothing. The judgement call is made; what remains is mechanical.

## Note

**The misreading that held this up is worth keeping.** The title says the VPS
holds no credentials that reach *upward*; it was read as the VPS holding no
credentials. A key on the server pointing at GitHub was permitted the whole time.
The next reader — human or agent — can make the same mistake and refuse a deploy
key it should be writing.

**The circularity is the reason this was filed rather than just decided.**
Tailscale SSH is the best answer to case 3, and it depends on the trust direction
being settled; the trust direction is most cleanly *expressed* in a Tailscale
ACL. Breaking the loop means deciding the principle first, on paper, and writing
the ACL to match.

**Writing a permissive ACL first and tightening later does not happen.** This is
the same shape as the firewall ordering in [[stage-3b-lockdown]], and the same
reason that item exists: a temporary permissive state with no forcing function
becomes permanent.

**The convenience being given up is real** and should not be argued away. Pulling
a file from the laptop while already SSH'd into the server genuinely requires
exiting and re-running from the laptop. That is a small tax paid continuously, in
exchange for a large loss avoided rarely — the correct shape for a security
trade, but not a free one.

**One asymmetry worth noticing.**
[[../../arch-thinkpad/decisions/20260818-devcontainer-agent-exposure|ADR-0003]]
already accepts unconditional agent forwarding into containers on arch-thinkpad.
Accepting that and refusing it here is defensible — a local container and a
public-facing host are different bargains — but it should be a stated distinction
rather than an inconsistency nobody noticed.

## Related

- [[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]]
- [[tailscale-ssh-decision]]
- [[stage-3b-lockdown]]
- [[../../arch-thinkpad/backlog/ssh-key-diversification|ssh-key-diversification]]
