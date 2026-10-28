---
type: decision
id: ADR-0032
aliases: [ADR-0032]
status: accepted
impact: medium
provenance: measured
device: arch-thinkpad
constrains: spec/base
stale: false
created: 2026-09-02T17:40:00Z
updated: 2026-09-02T17:40:00Z
last_verified: 2026-09-02
commit: pending
supersedes:
superseded_by:
tags: [decision, login, pam, kwallet, plasmalogin, authentication]
---

# ADR-0032 — The Plasma login path is accepted as installed

## Summary

`plasmalogin` authenticates through the same PAM stack as console login, with one
addition: the KDE wallet is unlocked by the login password. That is accepted; the
console login stays enabled as the fallback; autologin is never configured.

## Context

[[../backlog/login-path-unreviewed|login-path-unreviewed]] noted that
`plasma-meta` brought a display manager, and with it a PAM stack and an
auto-unlocking credential store, without anyone choosing them. Measured
2026-09-02 (`/usr/lib/pam.d/plasmalogin`, nothing overriding it in `/etc/pam.d/`):

- `auth`, `account`, `password` and `session` all `include system-login`, which
  is what `/etc/pam.d/login` includes — so `pam_faillock` (lockout after
  failures), `pam_nologin`, `pam_shells` and the rest apply identically.
- The greeter runs under `plasmalogin-greeter`, which is `pam_permit` on
  `auth` — it never authenticates anybody; the user's own session does.
- `pam_kwallet5.so` sits on `auth` and, with `auto_start`, on `session`. The
  wallet therefore opens with the login password, which means **wallet password
  and login password are the same secret**. This is what makes ADR-0019's
  Secret Service arrangement usable without a second prompt.
- No autologin: `/etc/plasmalogin*`, `/etc/sddm*` and `/etc/xdg/plasmalogin*`
  do not exist; the `plasmalogin-autologin` PAM file is present and unused.
- `getty@.service` remains enabled beside `plasmalogin.service`.

## Decision

Keep the stack as shipped. Keep `getty@` enabled as the recovery path. Do not
configure autologin: the LUKS passphrase then the login password are two
prompts on purpose. Accept that the wallet shares the login secret.

## Alternatives Considered

1. **Separate wallet password.** Two prompts per login on a single-user laptop
   behind full-disk encryption, defending against an attacker who already has a
   session as the user. Out of proportion to the threat model.
2. **Drop the display manager, start Plasma from the console.** Loses lock-screen
   and switch-user integration for no security gain — the PAM stack is the same.

## Consequences

### Positive

- One login, one secret, and the arrangement is now written down rather than
  inherited.
- The lockout and autologin questions are answered from files, not assumed.

### Negative

- Changing the login password without updating the wallet password breaks the
  auto-unlock silently: the wallet then prompts at login. That is the symptom to
  recognise, and `kwalletmanager` is where to re-sync.
- Lock and switch-user behaviour under `plasma-powerdevil` was **not**
  exercised; item 5 of the backlog record stays as a note there.

## Revisit when

A second user account appears on the machine, or the wallet starts holding
secrets that should not fall with the login password.

## Related

- [[20260828-secret-service-provider-is-ksecretd|ADR-0019]] — what the wallet is holding
- [[../backlog/login-path-unreviewed|login-path-unreviewed]] — closed by this
