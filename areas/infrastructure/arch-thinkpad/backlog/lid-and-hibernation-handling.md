---
type: backlog-item
status: resolved
priority: high
device: arch-thinkpad
projects: [power]
tags: [power, hibernate, lid, battery, noctalia, kde, powerdevil]
created: 2026-08-21T00:00:00Z
updated: 2026-08-28T00:00:00Z
resolved: 2026-08-28
commit: pending
---

> [!SUCCESS] Resolved 2026-08-28 by the desktop switch
> **`plasma-powerdevil.service` handles lid, lock and suspend natively, and they work
> out of the box.** No configuration was written. The service is loaded in the
> `graphical-session.target` graph in `state/services.md`. Exact timings are to be set
> in the KDE settings GUI later; the behaviour that was missing is present.
>
> **This item was blocked on two things that both stopped existing.** `hypridle` was
> gone, and Noctalia — which had taken over idle — exposed no `hibernate` verb. Neither
> is on the machine now; see
> [[../decisions/20260828-plasma-replaces-hyprland-userland|ADR-0018]].
>
> **The *Proposed* section below was never applied and is now moot.** Those
> `logind.conf` and `sleep.conf` edits would have put lid policy in one place while
> powerdevil claimed it in another. Do not apply them as written; powerdevil owns the
> policy, and a future change belongs in its settings.
>
> **What the item asked and this resolution does not answer.** Its second open question
> — whether recent systemd prefers battery-level estimation over `HibernateDelaySec`
> for `suspend-then-hibernate` — is untouched, because nothing here configured
> `suspend-then-hibernate` at all. The original symptom was *a bag overnight drains the
> battery flat*, and **that has not been re-tested** (`inferred`). Suspend working on
> lid close is not the same as hibernating before the battery is gone. If the flat
> battery recurs, this reopens as a hibernation-trigger question, not a lid question.

# Lid and hibernation handling

> [!NOTE] Summary
> Nothing handles the lid, and nothing hibernates on idle. A bag overnight still drains the battery flat despite hibernation working correctly when invoked. Recorded in Appendix C.5 as "the top backlog item" and still open.

## Context

Hibernation itself works — the swapfile is on `@swap`, `resume_offset=533760` is valid and survives a root rollback. What is missing is anything that *triggers* it.

**This item predates the Noctalia migration and needs revisiting.** The original plan was to change hypridle's 900s listener to `systemctl suspend-then-hibernate`, but `hypridle` was removed on 2026-08-19 and Noctalia owns idle now. Noctalia exposes `session <lock|suspend|lock-and-suspend|logout|reboot|shutdown>` — no `hibernate` verb.

## Proposed

`/etc/systemd/logind.conf`:

```
HandleLidSwitch=suspend-then-hibernate
HandleLidSwitchExternalPower=suspend
```

`/etc/systemd/sleep.conf`:

```
[Sleep]
HibernateDelaySec=30min
```

## Open questions

- Whether Noctalia's idle action can be pointed at `suspend-then-hibernate`, or whether a separate idle daemon is needed for that path.
- On recent systemd, `suspend-then-hibernate` prefers battery-level estimation over `HibernateDelaySec` where the hardware reports it. **Verify actual behaviour rather than assuming the 30-minute figure holds.**

## Related

- [[20260819-noctalia-userland-consolidation|ADR-0004]] — Noctalia owns idle
