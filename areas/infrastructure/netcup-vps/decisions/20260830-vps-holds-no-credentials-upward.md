---
title: The VPS holds no credentials that reach upward
type: decision
id: ADR-0027
aliases: [ADR-0027]
status: accepted
provenance: inferred
device: netcup-vps
stale: false
last_verified: 2026-08-31
commit: pending
impact: high
created: 2026-08-30
updated: 2026-08-31
tags: [decision, ssh, credentials, trust-model, tailscale, github]
constrains: spec/ssh, spec/tailscale
---

# ADR-0027 — The VPS holds no credentials that reach upward

- **Status:** **accepted** 2026-08-31 — see [[#Resolution]]
- **Date:** 2026-08-30 (proposed), 2026-08-31 (accepted)
- **Provenance:** inferred (the threat is reasoned, not observed)

## Summary

Proposed rule: **a machine may hold credentials to things below it in the trust
order, never above.** netcup-vps may reach GitHub repositories and the Hetzner
Storage Box; it must not hold a key that authenticates to arch-thinkpad or
pixel-graphene.

Filed as `proposed` because the operator has asked for something this rule
forbids, and the request is reasonable. Accepted 2026-08-31 once the git case
turned out not to require the forbidden thing at all.

## Context

The operator asked for a server-side SSH key covering three uses: `scp` to other
devices, git authentication, and "ssh into who knows what in the future."

The estate's existing model has trust flowing **inward**. Laptop and phone hold
private keys; servers hold `authorized_keys`. Devices under physical control
reach machines that are not. This is implicit rather than written down — it fell
out of the per-device key decision rather than being chosen — which is why it can
be inverted without anyone noticing.

A key on netcup-vps authorised on arch-thinkpad inverts it. And netcup-vps is the
worst candidate in the estate to hold such a key: it is the only machine with a
public IP, the only one running network services, and the one that will go weeks
without attention.

The concrete failure: someone with root on the VPS reads
`/root/.ssh/id_ed25519` — necessarily passphrase-free, because
[[20260830-no-luks-on-always-online-vps|ADR-0021]] establishes there is nobody
present to type one and an agent-held key on a server is decrypted in memory
anyway — and now has arch-thinkpad. From there: the restic repository, the
KeePassXC database file, this vault.

## Decision (proposed)

Trust order, most trusted first:

```
arch-thinkpad, pixel-graphene   →   netcup-vps   →   GitHub repos, Storage Box
```

A machine may hold credentials pointing rightward only.

## Alternatives Considered

For each of the three stated needs:

1. **File copy — invert the direction.** The server almost never needs to
   initiate. `scp` and `rsync` run from arch-thinkpad in either direction using
   the key that already exists. Covers the large majority of the actual need,
   requires no new credential, and is the recommended answer.
2. **Git — a per-repository deploy key, read-only.** Passphrase-free on the
   server, but scoped to one repository and revocable in that repository's
   settings without touching any other machine. The natural extension of the
   per-device model with the blast radius cut to one repo.
3. **Future unknowns — Tailscale SSH.** Authentication moves to tailnet identity
   with ACLs; there is no private key on the VPS to steal, revocation is central,
   and the ACL can express `arch-thinkpad → netcup-vps` while denying the
   reverse — an asymmetry a plain keypair **cannot represent**. Depends on this
   record being settled first, which is the circularity that makes both
   `proposed`.

Rejected outright:

4. **Agent forwarding (`ssh -A`) as a standing arrangement** — anyone with root
   on the VPS can use the forwarded socket to authenticate as the operator,
   anywhere, for the life of the session. Acceptable for a deliberate one-off,
   bad as a default. Note that
   [[../../arch-thinkpad/decisions/20260818-devcontainer-agent-exposure|ADR-0003]]
   already accepts unconditional agent forwarding into containers; extending the
   same exposure to a public-facing host is a different bargain.
5. **A general-purpose VPS key authorised everywhere** — the thing this record
   exists to prevent.

## Consequences if accepted

### Positive

- Compromise of the public-facing machine does not cascade to the endpoints that
  hold the vault, the backups and the password database.
- Revocation is bounded and knowable: each credential names one thing.
- The rule is expressible in a Tailscale ACL, so it can be enforced rather than
  merely documented.

### Negative

- **Some workflows get less convenient**, and the operator asked for the
  convenient version. Pulling a file from the laptop while SSH'd into the server
  requires exiting and running the copy from the laptop.
- A per-repo deploy key per repository is more objects to track than one key.
- If a general-purpose VPS key is adopted anyway, it must be its own key, never
  reused, never in an endpoint's `authorized_keys`, recorded in the KeePassXC
  inventory as a distinct trust tier, and constrained with `from=` wherever it is
  authorised. That is the mitigated form, not an exemption.

## Resolution

**Accepted as proposed, 2026-08-31.** No amendment; alternatives 1–3 stand as the
replacements for the three stated needs.

What moved it was a misreading being cleared rather than a trade-off being
accepted. The operator's objection was aimed at the git case — an SSH key on the
server felt like the thing the rule forbade. It is not: **GitHub sits to the
right of netcup-vps in the trust order**, so a key pointing at it is permitted and
always was. The rule constrains *direction*, not the existence of a key on the
host. Once that was explicit, the only convenience actually being surrendered was
case 1 — server-initiated `scp` to the endpoints — and the operator accepted it
without argument.

Worth keeping, because the same misreading is available to the next reader: *"the
VPS holds no credentials"* is not what this record says, and a future agent that
reads the title alone will refuse a deploy key it should be writing.

One step of alternative 2 is done: an ed25519 keypair was generated on the VPS
and its public half stored in KeePassXC (`measured`, 2026-08-31). Whether it has
been registered as a deploy key in the repository's settings is **not verified**
— tracked in [[../backlog/vps-trust-direction|vps-trust-direction]].

## Revisit when

Before the Tailscale ACL is written — the ACL is where the asymmetry gets
enforced rather than merely documented, and this record is now its premise.

**Pending move to `shared/`.** It constrains every device, not this one. Not moved
at acceptance because `shared/` is empty and five documents link to this path;
the move needs those links updated in the same commit. Tracked in
[[../backlog/vps-trust-direction|vps-trust-direction]].
