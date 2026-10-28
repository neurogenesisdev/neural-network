---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [security]
tags: [security, pam, login, kde, plasma, display-manager, authentication]
created: 2026-08-28T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Review the login path a metapackage changed

> [!NOTE] Summary
> `plasmalogin.service` is enabled where previously only `getty@.service` was. The machine gained a display manager — and with it a PAM stack, a session-setup path and an auto-unlocking credential store — as a side effect of installing `plasma-meta`. Nobody chose it and nobody has read it. This is the authentication surface, so "it works" is not the same as "it is understood."

> [!INFO] Steps 1–3 measured 2026-09-02; 4 and 5 remain
> **1. The PAM stack.** `plasma-login-manager` ships three files under
> `/usr/lib/pam.d/`, none overridden in `/etc/pam.d/`: `plasmalogin` (`auth`,
> `account`, `password`, `session` all `include system-login`, plus optional
> `pam_gnome_keyring` and **`pam_kwallet5`** on `auth` and `session auto_start`),
> `plasmalogin-greeter` (`pam_permit` — the greeter itself never authenticates),
> and `plasmalogin-autologin`. So it stacks exactly what console `login` stacks,
> through `system-login` → `system-auth` with `pam_faillock` `preauth`/`authfail`/
> `authsucc`, and adds the wallet hook.
> **2. The wallet is unlocked with the login password.** `pam_kwallet5.so` is on
> the `auth` line and `auto_start` on `session`; `kwallet-pam 6.7.4` is installed.
> That works only if the wallet password equals the user password, and it does
> work (ADR-0019), so **they are the same secret now**. Not decided anywhere.
> **3. Autologin: not configured.** No `/etc/plasmalogin*`, no `/etc/sddm*`, no
> `/etc/xdg/plasmalogin*`; the `plasmalogin-autologin` PAM file exists but
> nothing selects it. The LUKS-then-login sequence stands.
> **4 and 5** are decisions (keep both login paths; lock and switch-user
> behaviour) and were not taken.

> [!SUCCESS] Resolved 2026-09-02 by [[../decisions/20260902-plasma-login-path-accepted|ADR-0032]].

## Context

Measured from `state/services.md`, re-collected 2026-08-27: `plasmalogin.service`
is the **only** addition to the system-enabled unit list across the desktop
switch, and `getty@.service` is still enabled alongside it. So the console login
path was not replaced; a graphical one was added in front of it.

**This is the same class as [[undeliberate-listeners-review]].** That item records
three daemons listening that nobody deliberately enabled, and its finding was that
the assumption *"nothing listens that I did not put there"* is false. The
equivalent assumption here is *"nothing authenticates me except the thing I set
up"*, and it is false in the same way and for the same reason: a metapackage
brought a component, the component shipped a unit, and the unit was enabled by the
package rather than by a person.

The difference is consequence. An unreviewed listener behind a default-drop
firewall is contained. An unreviewed **PAM stack is the authentication path
itself**, and it now also gates a credential store — see
[[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]], where
`plasma-kwallet-pam.service` is the mechanism that unlocks the wallet at login.
The login path and the secret store became one subject without either being decided.

## Steps

1. Read the PAM configuration `plasmalogin` installed. Establish what it stacks,
   in what order, and what it does that `login` did not.
2. Establish whether `pam_kwallet` unlocks the wallet with the login password, and
   what that implies — specifically whether the wallet password and the user
   password are now the same secret. This is the item's most likely real finding.
3. Check for autologin. A display manager makes it a one-checkbox change, and it
   would silently defeat the LUKS→login sequence as a two-factor arrangement.
4. Decide whether both login paths should stay enabled. `getty@.service` remaining
   is almost certainly correct (it is the recovery path), but it should be a
   decision rather than a leftover.
5. Confirm what happens to the session on lock and on switch-user, now that
   `plasma-powerdevil` owns lock — see the resolved
   [[lid-and-hibernation-handling]].

## Note

**The trap is treating this as a cosmetic change.** The visible difference is a
graphical login screen instead of a text one, which reads as a theme. The actual
difference is a new PAM stack running as root before any session exists, plus a
credential store that unlocks automatically as a consequence.

`pam_faillock` behaviour is worth re-checking here too. The
[[areas/infrastructure/arch-thinkpad/state-notes|notes.md]] entry on it is `inherited-unverified` and describes the
*console* symptom ("will look identical at the login prompt"). A display manager
may present lockout differently or not at all, which would make an already
confusing failure worse.

## Related

- [[../decisions/20260828-secret-service-provider-is-ksecretd|ADR-0019]] — the DM is what unlocks the wallet
- [[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]] — the switch that brought it
- [[undeliberate-listeners-review]] — same class, smaller blast radius
- [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]] — `pam_faillock`, inherited and unverified
