---
type: backlog-item
status: active
priority: medium
device: netcup-vps
projects: [netcup-build]
tags: [backlog, security, tailscale, ssh, acl]
created: 2026-08-30
updated: 2026-08-31
commit: pending
---

# tailscale-ssh-decision

> [!NOTE] Summary
> Whether to enable Tailscale SSH on this host, replacing key-based auth over the
> tailnet with tailnet-identity auth governed by ACLs. **Unblocked 2026-08-31**:
> its premise, [[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]],
> is now accepted, so the asymmetry an ACL would express is settled policy rather
> than an assumption. Ready to be decided and written up.

## Context

The question came up while planning stage 3b, because it changes what "bind sshd
to the Tailscale address" means.

**With Tailscale SSH:** authentication is tailnet identity plus ACL. No
`authorized_keys` to maintain, central revocation, an audit trail, and — the
property that matters for ADR-0027 — the ACL can express *arch-thinkpad may reach
netcup-vps, netcup-vps may not reach arch-thinkpad*. A plain keypair cannot
represent that asymmetry at all.

**Without it:** key auth continues to work, unchanged, over the tailnet. One
fewer component in the authentication path, and the keys are already working.

## Steps

1. ~~Settle [[vps-trust-direction]] / ADR-0027~~ — done, accepted 2026-08-31.
2. Decide this, and write it up as an ADR if the answer is yes — it constrains the
   ACL, the sshd config, and the key inventory.
3. If yes: enable with `tailscale up --ssh`, write the ACL, and **verify the
   asymmetry actually holds** by attempting the forbidden direction and confirming
   it fails. An ACL that has not been tested in the denying direction has not been
   tested.
4. Decide whether key auth stays as a fallback or is removed. Keeping both means
   the ACL is not actually the control.

## Blocked on

Nothing. ADR-0027 is accepted and the asymmetry it establishes is what an ACL
would express.

## Note

**One argument shifted since this was filed.**
[[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]]
declined to bind sshd to the Tailscale address specifically to avoid making SSH
access depend on `tailscaled` being up at boot. Tailscale SSH deepens that
dependency rather than avoiding it — access would require `tailscaled`, the
control plane for *authorisation* decisions, and a correct ACL. The same
reasoning that closed step 6 argues against enabling it, or at minimum against
removing key auth if it is enabled. That tension is now the decision, and it is
not the one this item originally framed.

**The strongest argument for it is the one that has nothing to do with
convenience.** It removes a private key from the machine most likely to be
compromised. That is the same reasoning as ADR-0027 and it is why the two are
coupled.

**The argument against is dependency depth.** SSH access would then depend on
`tailscaled` running, the tailnet control plane being reachable, and the ACL being
correct. Key auth over the tailnet depends on `tailscaled` and the tailnet, but
not on the control plane for *authorisation* decisions. For a machine checked
monthly by one person, a shallower dependency chain has real value — and the
console fallback exists precisely because remote auth can fail.

**Do not enable it as an experiment during stage 3b.** That step already has two
lockout-capable changes in it; adding a third authentication mechanism mid-lockdown
means three candidate causes for one symptom.

## Related

- [[vps-trust-direction]] — the blocker
- [[../decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]]
- [[stage-3b-lockdown]]
