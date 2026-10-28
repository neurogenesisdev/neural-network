---
type: index
device: arch-thinkpad
updated: 2026-08-30
tags: [index]
---

# Runbooks

Goal-directed procedures, entered at need rather than read start to finish. The
build spec tells you how the machine was made; a runbook tells you what to do
when one part of it stops holding.

| Runbook | Fires when | Urgency |
|---------|-----------|---------|
| [[tpm2-reenroll\|Re-enroll TPM2 after a firmware update]] | Boot asks for the LUKS passphrase instead of the TPM PIN | None — the machine is usable; only the convenience path is gone |
| [[bootloader-reinstall\|Reinstall the bootloader without unsigning it]] | The named firmware boot entry is gone, **or** the firmware refuses the loader and there is no menu | None for the first; high and needs the Arch ISO for the second |
| [[restic-restore\|Restore from restic]] | A file or tree is lost and snapper cannot supply it — or the machine itself is gone | Low for a file; the machine case is the one to have rehearsed |

The first two fire on a firmware update, and a firmware update can produce both at
once. `bootloader-reinstall` says so at the end.

**`restic-restore` is `inferred` and says so at the top** — the only runbook here
that has never been executed even once. The other two were written from procedures
that had been performed; this one is written from restic's help output against
repositories nothing has ever restored from. Whoever uses it first is testing it,
which is the whole of
[[../backlog/restic-restore-untested|restic-restore-untested]]. A runbook read for
the first time during the incident it covers is a worse artifact than one rehearsed
once on something unimportant.

## Single-sourcing

Runbooks and the build spec share step definitions rather than duplicating
them — in principle. In practice **both runbooks duplicate their verification
blocks, for two different reasons**, and the reasons are the useful part.

The mechanism is the constraint: a step record is one fenced block, so
`![[../spec/boot#tpm2-enroll]]` brings `command:` along with `postcondition:`.

- **`tpm2-reenroll` duplicates because the spec's command is dangerous here.**
  It is the *first* enrollment, with no `--wipe-slot=tpm2`; run from the runbook
  it would add a second token to a volume that already has a stale one. Recorded
  in [[../sessions/20260823-spec-boot-promotion|the 2026-08-23 log]].
- **`bootloader-reinstall` duplicates although the spec's commands are safe.**
  It needs them in a fixed order, gated on a diagnosis, with a chroot variant no
  step record covers. Transclusion would carry prose arguing for a booted-system
  context onto a page read at a firmware error screen.

The general limit is in [[../spec/_index|spec/_index.md]]. Neither case is an
argument against transclusion as such; the question is what each option costs
when it fails, and a runbook is read under conditions where a wrong adjacent
command is expensive.

## What belongs here

- A procedure with a trigger, entered under pressure and often on a phone
- Something that will recur: `tpm2-reenroll` fires on every firmware update
- Anything where the correct first move is *stop and diagnose* rather than *fix*

Diagnostics whose error message points away from the cause go in
[[areas/infrastructure/arch-thinkpad/state-notes|notes.md]], not here. A decision with a rationale goes in
[[../decisions/_index|decisions/]].
