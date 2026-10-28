---
type: backlog-item
status: resolved
priority: low
device: arch-thinkpad
projects: [security]
tags: [luks, encryption, recovery, single-point-of-failure]
created: 2026-08-21T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# Add a second LUKS keyslot

> [!NOTE] Summary
> One passphrase into one header. `cryptsetup luksAddKey` costs nothing.
> **Repriced high → low on 2026-08-31**: the risk it guards is already retired by
> the operator's passphrase custody, not by anything in this vault. Still worth
> doing eventually; not worth doing next.

> [!SUCCESS] Resolved 2026-09-03 — `measured`
> Keyslot **1** added (argon2id), beside 0 (passphrase) and 2 (TPM2+PIN). Header
> backup re-taken with all three slots: `/root/luks-header-nvme0n1p2-20260903.img`
> and a copy in the operator's home for syncing to Proton Drive and pixel-graphene;
> the veracrypt vault was mounted read-only at the time, so no copy landed there.
> Steps 1–4 done. **Step 5 measured 2026-09-03** by `cryptsetup open --test-passphrase
> --key-slot 1`: `Command successful`. A normal boot never asks for a passphrase —
> the TPM PIN path answers first — so the on-system test is the meaningful one.
>
> **A first attempt added the wrong passphrase.** The new passphrase was on the
> clipboard from KeePassXC, which clears it after a short timeout; by the time
> `luksAddKey` prompted, the clipboard held the command line itself, and that
> string became the key. `luksDump` cannot show it — a keyslot looks the same
> whatever it holds. Removed with `luksKillSlot 1` (which asks for a passphrase
> from any *other* slot, so the real one was never at risk) and redone after
> pasting into a text field first to see what the clipboard held. Rule for any
> secret typed twice from a clipboard: **paste it somewhere visible first.**

## Context

The LUKS2 container has one passphrase keyslot plus the TPM2+PIN enrollment. A forgotten or mistyped passphrase, combined with any event that invalidates the TPM binding, means the disk is unrecoverable.

The header backup exists in three places (Pixel, Proton Drive, Ventoy `tools/`), but a header backup restores the *header*, not a key you do not know.

## Repriced 2026-08-31

The framing above assumes the passphrase can be lost. **It cannot easily be**, and
this vault had no way to know that. Stated by the operator (`measured` only in the
sense that he reported it; nothing here verifies KeePassXC's contents):

- The passphrase is memorised, and used often enough to stay memorised.
- It is in KeePassXC.
- That database syncs to pixel-graphene via Syncthing, and is backed up to Proton
  Drive.

That is three independent copies plus recall. A second keyslot insures against
**forgetting or corrupting the one passphrase**; the header backup insures against
**losing the header**. Both risks are already covered by other means, so the item
buys redundancy on top of redundancy.

**The general failure is worth more than the specific correction.** This item sat
at `high` because it was filed under *lockout*, and lockout items inherit urgency
from the word. The vault cannot see what is in KeePassXC or what the operator has
memorised, so **it systematically overprices any risk whose mitigation lives
outside the machine.** Other items filed on recovery grounds deserve the same
question asked out loud: *what already covers this that the vault cannot see?*

It stays open rather than closed because the argument is about priority, not
value: `luksAddKey` is still cheap, and a keyslot that exists is better than one
that does not.

## Steps

1. `sudo cryptsetup luksAddKey /dev/nvme0n1p2`
2. Store the second passphrase in KeePassXC, distinct from the first.
3. `sudo cryptsetup luksDump /dev/nvme0n1p2` — confirm the new keyslot.
4. **Re-take the header backup** — the old one does not contain the new slot.
5. Test the new passphrase at an actual boot before trusting it.

## Note

Step 4 is the one that gets skipped. A header backup taken before `luksAddKey` will restore a header without the second key.
