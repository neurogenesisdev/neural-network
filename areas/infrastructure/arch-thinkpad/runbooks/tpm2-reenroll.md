---
title: "Re-enroll TPM2 after a firmware update"
type: runbook
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-10-03
commit: pending
created: 2026-08-23
updated: 2026-10-03
tags: [runbook, tpm2, luks, firmware, secure-boot]
---

# Re-enroll TPM2 after a firmware update

**This is expected, not a fault.** The enrollment binds PCRs 0+2+7
([[../decisions/20260823-tpm2-pcr-selection|ADR-0013]]). PCR 0 is "core system
firmware executable code; changes on firmware updates" — so every firmware
update invalidates the TPM unlock by design. That is the property being paid
for, not a malfunction.

**Firmware updates are not the only trigger.** A change in *what early boot
measures* moves the same PCRs with no firmware update involved. Seen once,
2026-10-03: a `pacman -Syu` that took mkinitcpio 41.1 → 42.2 and systemd
261.2 → 262 (*measured*, `/var/log/pacman.log`). The Arch news item of
2026-09-22 (read 2026-10-03) says that from mkinitcpio 42 the systemd hook
ships `systemd-pcrosseparator.service`, which changes the measurements of PCRs
0-7, 9 and 12-14 and requires re-enrollment. That this was the whole cause is
*inferred*: it fits every observation below and nothing was run to isolate it.
A kernel upgrade by itself, or a `linux-firmware` update — driver files the
kernel loads, not the system firmware PCR 0 measures — should not do this
(*inferred* from the PCR semantics in ADR-0013).

## How it presents

The boot asks for the TPM PIN as usual, pauses, and then asks for the **LUKS
passphrase** as well (*measured* 2026-10-03). Do not retry the PIN. Nothing is
printed at the prompt to say why, but the journal has it: `journalctl -b -u
systemd-cryptsetup@cryptroot` shows `TPM policy does not match current system
state` (*measured*). The system boots normally once the passphrase is entered.

*Corrected 2026-10-03: this section said the passphrase prompt replaces the PIN
prompt and that nothing is logged. Both were written before the runbook had
ever fired.*

Enter the passphrase and continue. There is no urgency — the machine is fully
usable; only the convenience path is gone.

## Confirm it is the expected cause

Before re-enrolling, confirm a PCR moved and that it moved for a reason you
recognise. Re-enrolling blindly against a *compromised* firmware state seals the
volume to the attacker's boot chain, which is the one way to turn this
inconvenience into a real loss.

```bash
# Was there a firmware update at all?
fwupdmgr get-history
# What changed. state/ is a snapshot: the diff is empty until the collector has
# run again on the current boot (measured 2026-10-03).
sudo systemd-analyze pcrs | awk '$1 ~ /^(0|2|7)$/ {print $1, $2, $3}'
(cd ~/dev/infra-docs/arch-thinkpad && ./collect/collect.sh)
git -C ~/dev/infra-docs --no-pager diff HEAD -- arch-thinkpad/state/boot.md
```

Do not commit that collector run yet — re-run it after re-enrolling and commit
once, so one diff records the whole event.

- **PCR 0 moved, and you ran a firmware update** — expected. Proceed.
- **PCRs 0, 2 and 7 all moved, right after a `pacman -Syu`, and `fwupdmgr
  get-history` shows no update** — the second trigger above (*measured*
  2026-10-03: `No history`, all three changed). Find the archlinux.org/news
  item that explains it before proceeding; if there is none, treat it as the
  next case.
- **PCR 0 moved alone, and you did not run a firmware update** — stop. Do not
  re-enroll. SPI write protection is disabled on this hardware (ADR-0013), so
  firmware modification is reachable from a root shell; an unexplained PCR 0
  change is exactly the event the binding exists to catch.
- **PCR 7 moved** — Secure Boot state or a certificate database changed. A `dbx`
  revocation update delivered through fwupd does this. Check `sbctl status`
  still reports Secure Boot enabled and Setup Mode disabled.
- **PCR 2 moved** — unexplained. ADR-0013 records PCR 2 as measured non-empty
  but not understood, and its stability across peripheral attachment is untested.
  If this recurs on hotplug rather than on firmware updates, ADR-0013 needs
  revisiting rather than the enrollment repeating.

## Re-enroll

One invocation, not two. `systemd-cryptenroll(1)`: when a wipe is combined with
an enrollment, "the enrollment is completed first, and only when successful the
wipe operation executed — and the newly added slot is always excluded from the
wiping." Wiping first and enrolling second would leave the volume
passphrase-only if the enrollment failed.

```bash
luksdev=$(sudo cryptsetup status cryptroot | awk '/device:/ {print $2}')
sudo systemd-cryptenroll \
  --wipe-slot=tpm2 \
  --tpm2-device=auto \
  --tpm2-pcrs=0+2+7 \
  --tpm2-with-pin=yes \
  "$luksdev"
```

The passphrase keyslot is untouched — `--wipe-slot=tpm2` wipes only "all key
slots that are unlocked by a TPM2 chip." `systemd-cryptenroll` additionally
refuses as "a safety precaution an operation that wipes all slots without
exception."

**The passphrase remains the only guaranteed way in.** Keep it memorised. Every
firmware update breaks the TPM path until this runbook is run.

**Since systemd 262 the same command enrolls a different kind of PIN slot.**
Run 2026-10-03 it printed `Argon2id benchmark: 2 iterations, 3835 MiB, 12
lanes, ~1968ms`, then `New TPM2 token enrolled as key slot 3. Wiped slot 2.`
(*measured*). The slot it replaced reported `tpm2-argon2id: false`
(*measured*, `cryptsetup luksDump`). The PIN step at boot takes about two
seconds longer. The 262 release notes name `--tpm2-with-pin=direct` as the old
behaviour (*inherited-unverified* — not run here). The TPM keyslot is now 3,
not 2; `backlog/luks-second-keyslot` still says 2.

## Verify

Duplicated from [[../spec/boot#tpm2-enroll|spec/boot#tpm2-enroll]], deliberately.
**Both copies must move together.**

```bash
luksdev=$(sudo cryptsetup status cryptroot | awk '/device:/ {print $2}')
if [ -z "$luksdev" ]; then
  echo "RESULT=FAIL (cryptsetup status named no backing device)"
else
  sudo cryptsetup luksDump --dump-json-metadata "$luksdev" \
    | jq -e '[.tokens[] | select(.type == "systemd-tpm2")] as $t
             | ($t | length) == 1
               and ((($t[0]["tpm2-pcrs"] // []) | sort) == [0, 2, 7])
               and ($t[0]["tpm2-pin"] == true)' >/dev/null \
    && echo RESULT=PASS || echo RESULT=FAIL
fi
```

A token existing proves nothing on its own: `--tpm2-pcrs=` defaults to binding
to no PCRs at all, and an unbound enrollment is indistinguishable from a bound
one by presence.

**The `sudo` on both `luksdev=` lines was added 2026-10-03, and the two copies
no longer match.** As a normal user `cryptsetup status cryptroot` prints
`Cannot initialize device-mapper, running as non-root user.` and returns
nothing, so this block reported `RESULT=FAIL (cryptsetup status named no
backing device)` on a correct enrollment; with `sudo` it printed `RESULT=PASS`
(both *measured*). The `-z` guard did its job — it named the real problem
rather than blaming the enrollment. `spec/boot#tpm2-enroll` was **not**
changed: its step carries no `sudo` anywhere, and its mutation set was not
re-run. It is recorded under that chapter's open items.

The `-z` guard is the 2026-08-24 sweep reaching this page: with an empty
`$luksdev`, `luksDump` fails and the `||` reports `RESULT=FAIL`, which reads as
"the enrollment is wrong" when the truth is "there is no mapping to look at" —
and this page is read at an unexpected passphrase prompt, where that is the
worst available misdirection. `spec/boot#tpm2-enroll` carries the same guard.

**Why this is not transcluded, now that `spec/boot` exists.** The obvious move
is `![[../spec/boot#tpm2-enroll]]`, and it was written that way first. It is
wrong. A step record is one fenced block, so transcluding the section brings
`command:` with it — and that command is the *first* enrollment, with no
`--wipe-slot=tpm2`. Run here it would add a second TPM2 token to a volume that
already has a stale one, and the assertion above would then fail for a reason
the reader had just created.

A warning beside it is not enough. This page is read at an unexpected passphrase
prompt, which is the worst condition under which to ask someone to notice that
one of two adjacent commands is the wrong one. Targeting just the
`postcondition:` would mean splitting the step record into separate fences,
which is schema churn driven by a consumer rather than by the spec.

So the drift risk `_schema.md` warns about is accepted here, with eyes open: it
is one `jq` expression, and it is cheap to compare. The risk it is being traded
against is a destroyed working enrollment. Single-sourcing is a means, not
the goal.

Then re-run the collector so `state/` reflects the new PCR values, and commit —
the diff is the record that this happened.

```bash
cd ~/dev/infra-docs/arch-thinkpad && ./collect/collect.sh
```

Reboot to confirm the PIN prompt returns. Until you have, the TPM path is
unverified.

## Red `[FAILED]` lines that are not this problem

From systemd 262 the boot prints `[FAILED] Failed to start Early TPM SRK
Setup` and `[FAILED] Failed to start TPM NvPCR Product ID Measurement`. The
journal lines behind them are `Failed to initialize NvPCR index: No such file
or directory` and `Could not extend NvPCR: No such file or directory`
(*measured* 2026-10-03, `journalctl -b -u systemd-tpm2-setup-early -u
systemd-pcrproduct`). They appeared on the same boot as the passphrase
fallback and are a separate matter: they were still printed on the next boot,
where the PIN alone unlocked the volume (*measured*). They say nothing about
the enrollment.

The 262 release notes (read 2026-10-03) give the reason: NvPCRs now need their
definitions and a signed initrd PCR policy shipped in the UKI, and "will fail
to initialize" otherwise. These UKIs are built by mkinitcpio and carry neither.
Upstream issue systemd/systemd#43848 tracks it, open as of 2026-10-03.
