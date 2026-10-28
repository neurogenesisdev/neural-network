---
title: The Secret Service provider is ksecretd, and the handover was unannounced
type: decision
id: ADR-0019
aliases: [ADR-0019]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-27
commit: pending
impact: high
created: 2026-08-28
updated: 2026-08-28
tags: [decision, kde, kwallet, keepassxc, secret-service, credentials, security, dbus]
constrains: spec/desktop
---

# ADR-0019 — The Secret Service provider is ksecretd, and the handover was unannounced

- **Status:** accepted — ratification of a change already in effect, **conditional**;
  see *Revisit when*
- **Date:** 2026-08-28
- **Provenance:** measured — bus ownership, the greyed-out KeePassXC setting and the
  process list are the operator's readings on 2026-08-27, taken on the host. None of
  them is reproducible from `state/`; the reason is recorded below. The history in
  *What happened to the secrets already in the old store* is
  `inherited-unverified` from `_source/`.

## Summary

`org.freedesktop.secrets` is held by `ksecretd`. It took the name when Plasma was
installed. **Nothing announced the change** — no prompt, no error, no choice
presented — and it was discovered afterwards by noticing that a setting in
KeePassXC had gone grey.

## Context

This record is written as an unannounced change discovered afterwards, **not as a
decision taken**, because that is what happened and because the sequence is the
part worth keeping.

### What is true now

- `org.freedesktop.secrets` is held by **`ksecretd`**, with
  `org.kde.secretservicecompat` alongside it.
- **KeePassXC's Secret Service setting is greyed out in its own UI.** It was not
  switched off by anyone. KeePassXC detects that another provider already holds the
  name and declines rather than competing.
- `plasma-kwallet-pam.service` is in the `graphical-session.target` graph in
  `state/services.md`, and `plasmalogin.service` is enabled where previously only
  `getty@.service` was. **The display manager is the mechanism that unlocks the
  wallet** — the login path and the credential store are one subject, which is why
  plasmalogin is recorded here.

### KWallet won uncontested, not by a race

Only `keepassxc-proxy` was running at the time. There was no contention to
resolve, so nothing arbitrated between two live providers.

This is stated explicitly because the natural assumption — "whoever asked first
won, so restarting things in the right order would flip it back" — is wrong, and
acting on it would waste a session. The handover happened because one provider was
absent from the bus at the moment the other claimed it.

## Decision

Accept `ksecretd` as the sole Secret Service provider. Do not contest it, do not
re-enable KeePassXC's integration, and do not attempt to order the two providers'
startup.

**This acceptance is conditional.** The condition is in *Revisit when*, not here,
and it is not rhetorical — see the note attached to it.

## Rationale

Contesting the name means arranging for KeePassXC to claim it before `ksecretd`
does, on every login, with the loser silently declining. That is a startup-ordering
race as a security control, on a machine where the winner determines where
credentials are written. Losing the race quietly is the failure mode, and it looks
identical to working.

The greyed-out setting is also, read correctly, KeePassXC behaving well: it refuses
to fight for a name another process holds. Building a mechanism to defeat that is
building around a deliberate safety behaviour.

What is *not* a rationale: that KWallet is better than KeePassXC for this. That
comparison was never made, by anyone, and this record should not be cited as if it
had been.

## Consequences

### What happened to the secrets already in the old store

The named consumer was VS Code. `_source/Block-C-Dev-Environment.md` §3.8 set
KeePassXC up as the provider specifically for it, exposing a dedicated
`SecretService` group, and forced VS Code onto libsecret with
`{ "password-store": "gnome-libsecret" }` in `~/.vscode/argv.json`.

**All of that is `inherited-unverified`.** `_source/` is an input being refactored
away, not documentation, and the root `CLAUDE.md` forbids grounding a claim in it.
It is recorded here as the historical configuration to re-read on the host, not as
current fact.

Three states, marked separately:

1. **VS Code followed the handover without noticing, and that is the mechanism**
   (`inferred`). §3.8 says in as many words that `gnome-libsecret` "does not require
   gnome-keyring — it means *use libsecret against whatever owns
   `org.freedesktop.secrets`*." The setting names a **bus name, not a provider**, so
   the client kept working across a provider swap. That is precisely why nothing
   announced itself: the only component configured to care was configured in a way
   that could not tell the difference. Lookup: `~/.vscode/argv.json`.
2. **The KeePassXC entries are orphaned, not lost** (`inferred`). KeePassXC declined
   to serve the bus; it deleted nothing. Secrets written through it before the switch
   still sit in the `SecretService` group in the KDBX and are simply no longer
   reachable by any Secret Service client, while VS Code re-authenticates and writes
   fresh secrets into KWallet. **The machine now has two credential stores, one of
   them inert.** Nothing here can see either.
3. **Storage properties changed, and this was the part nobody was offered.** §3.8
   notes that a token in the KDBX "syncs to the Pixel and Proton Drive," and that
   "when the lock timer fires, VS Code loses the token and re-prompts." Neither
   transfers. A secret in KWallet is local to this machine and is unlocked by PAM at
   login rather than by a timer. So the same credential moved from *synced,
   timer-gated* to *local, unlocked-for-the-session* without anything being decided.
   Whether that is better or worse depends on which failure is being insured against
   — the point is that it was a trade made silently.

### The mitigation that justified the original arrangement may not have survived

§3.8's argument for tolerating Secret Service exposure at all was a specific
control: "Keep *Confirm when passwords are retrieved by clients* ticked — **it is
the per-request prompt that makes exposure tolerable**." Everything in an exposed
group is readable by any client, and the prompt was what bounded that.

**Whether KWallet offers an equivalent once `plasma-kwallet-pam` has unlocked it at
login is not established here, and is not guessed at.** If it does not, the handover
traded a per-request confirmation for an unlock-once wallet readable by every
process running as the user — which is a material change to the credential exposure
model, arrived at without a decision. This is the condition in *Revisit when*.

Lookups that close it, both on the host: `kwalletmanager`'s per-application access
control, and `kwalletrc`.

### `ksshaskpass` is a second askpass path, and is flagged rather than resolved

`ksshaskpass` is installed (operator, 2026-08-27). That is a second askpass
mechanism alongside the existing KeePassXC SSH-agent arrangement in
[[20260818-devcontainer-agent-exposure|ADR-0003]] and the
`IdentitiesOnly` / `ssh-add -l` entries in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]].

Which one services a given prompt is **not established** (`inferred`). Deliberately
not resolved in this round: it interacts with agent forwarding into containers, and
guessing at it would put a plausible wrong answer next to the SSH entries in
`notes.md` that exist because of an earlier plausible wrong answer.

The lookup that closes it: `SSH_ASKPASS` and `SSH_ASKPASS_REQUIRE` in the session
environment, read from a graphical session rather than from a TTY.

### The vault cannot reproduce most of this

Of the four load-bearing readings — `ksecretd` holding the bus name, the greyed-out
KeePassXC setting, `ksshaskpass` being installed, and `plasmalogin.service` being
enabled — **only the last is in `state/`**. The first three had to be measured on
the host and cannot be re-checked from the vault. See
[[../backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]].

## Revisit when

**If `kwalletmanager`'s per-application access control turns out not to offer a
per-request prompt equivalent to KeePassXC's *Confirm when passwords are retrieved
by clients*, this ratification is reopened.** The exposure model changed without a
decision, and that is the reading which decides whether accepting it was right.

**If KeePassXC's Secret Service setting stops being greyed out**, `ksecretd` has
released the name, the uncontested-win reasoning no longer describes the machine,
and the choice becomes a real one.

> These are conditions that reopen the record, not things to bear in mind. The
> distinction has cost this vault once already:
> [[20260816-container-storage-and-bind-mounts|ADR-0006]] put "harmless while
> empty; tighten if it gains content" in a Consequences bullet, the condition
> fired, and nobody noticed until
> [[20260825-harness-state-and-identity|ADR-0017]] audited it two months later and
> recorded that "the condition the ADR set has been met and nothing happened."
