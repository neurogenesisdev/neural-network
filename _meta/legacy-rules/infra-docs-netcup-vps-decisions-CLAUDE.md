# decisions/ — append-only

Same rule as `arch-thinkpad/decisions/`. To reverse a decision, write a new
record and set `superseded-by` on the old one. **Never edit a record's Decision
or Rationale after acceptance**, and never delete a file — the history of a
reversed decision is why it does not get reversed back.

Amendments that add cost or consequence are fine; mark them with an `Amended:`
line. New records use `../../_templates/tpl-decisions.md`.

## ADR ids are global, not per-device

arch-thinkpad holds ADR-0001…ADR-0020 and ADR-0028. This folder holds
ADR-0021…ADR-0027 and ADR-0029.
A new record here takes the next free number across the whole vault, not the next
number in this folder. See `../_index.md` for why.

## Three records here are load-bearing and easy to re-litigate by accident

- **ADR-0021 rejects full-disk encryption**, and the reasoning is specific to an
  always-online host — do not propose LUKS for this machine on the grounds that
  arch-thinkpad has it. The two are different threat models and the record says
  so. Its *negative consequences* carry a standing obligation: no long-lived
  secret in plaintext on disk, which constrains every service config written for
  this host.
- **ADR-0027 constrains the *direction* a credential points, not whether a key
  may exist here.** Accepted 2026-08-31. GitHub and the backup target sit *below*
  this host in the trust order, so a key authenticating to them is permitted and
  a per-repository deploy key is the intended pattern. What is forbidden is a key
  that authenticates to arch-thinkpad or pixel-graphene. The title reads like a
  blanket prohibition and has already been misread once — do not refuse a deploy
  key on its authority.
- **ADR-0029 makes nftables the single control on SSH reachability.** sshd binds
  the wildcard deliberately. Do not propose `ListenAddress` on the Tailscale
  address as a hardening improvement; the record explains why the boot-order
  dependency is the worse trade on this machine, and hive already paid for the
  adjacent `ssh.socket` version of it.

## No `state/` to ground claims against

The vault's central rule is that factual claims resolve against collector
output. **This device has no collector**, so every claim in this folder is
`measured` at a point in time with nothing re-verifying it. Mark provenance
honestly and prefer re-reading the host over trusting a file. See
`../backlog/netcup-collector.md`.
