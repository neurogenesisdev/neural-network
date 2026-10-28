# build/ — destructive scripts, tokenised values

These three scripts built the machine. `stage1-partition-bootstrap.sh` wipes
`/dev/vda` unconditionally once its guards pass.

## Never inline real site values

Every script opens with a `# --- site values ---` block holding placeholders —
`<NETCUP_IPV4>`, `<NETCUP_IPV6>`, `<SERVER_ID>`. **The committed scripts do not
run as-is**, by design. Substitute from `redact.map` at the point of use and
never commit the substituted copy.

**Nothing enforces this.** The pre-commit hook's pattern set covers UUIDs, MACs,
Tailscale ranges and machine-id shapes; a routable public address is none of
those and will pass the gate silently. The discipline is the only control here.

## Do not weaken the guards

Two are structural rather than stylistic:

- `stage1` refuses unless `/` is an `overlay` mount, so an accidental run on the
  installed system or on arch-thinkpad aborts instead of wiping a disk.
- `CONFIRM=WIPE` is deliberately not defaultable.

`stage3` asserts its own nftables policy is still `accept`. That check exists to
fail loudly if someone tightened the firewall without passing the Tailscale gate
in `../backlog/stage-3b-lockdown.md`. Do not "fix" it to expect `drop`.

## Postconditions assert against generated artifacts

`net.ifnames=0` is checked in `/boot/grub/grub.cfg`, not `/etc/default/grub` —
asserting on the input would pass while `update-grub` silently failed. fstab is
built from `blkid` at run time rather than from recorded UUIDs. Keep both
properties in any edit; they are the root `CLAUDE.md`'s rule about constants in
postconditions, applied.

## Stage 3b is absent on purpose

The lockdown is not written because its safe execution depends on a condition
the script cannot check — that tailnet SSH has been verified from arch-thinkpad.
Writing it in advance would produce a file that looks runnable and is not. See
`../backlog/stage-3b-lockdown.md`.

## The committed stage 1 is not what ran

It is the corrected version — it creates `@cache`, which the 2026-08-30 run did
not. That is right for a script whose purpose is to rebuild the machine and wrong
as a historical record; the history is in `../sessions/`.
