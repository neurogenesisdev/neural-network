---
type: index
device: arch-thinkpad
updated: 2026-09-02
tags: [index]
---

# State

**Generated — do not hand-edit.** `notes.md` is the only hand-written file here.

| File | Covers |
|------|--------|
| packages.md | Explicit, AUR, orphans, modified packaged configs, bounded presence list of packages that records depend on, tool versions |
| services.md | Enabled system and user units, `graphical-session.target` graph |
| hardware.md | CPU, memory, PCI, USB, battery design capacity, charge thresholds |
| boot.md | bootctl, cmdline, UKIs, mkinitcpio, sbctl, kernel versions |
| storage.md | Block devices, LUKS, LUKS tokens, crypttab, subvolumes, default subvolume, mounts, fstab, swap, snapper, NVMe health |
| network.md | Interfaces, permanent MACs, nftables, listening sockets, NM config, usbguard |
| harness.md | Agent-harness config extracted from `.claude.json` — per-project trust, MCP servers, `allowedTools`; top-level listing of the two agent state directories beside the live restic exclusion list |
| collector-run.md | One run marker: UTC timestamp and collector git revision. The only line meant to change on every run — all other files are published together from a staging directory, so a partial run changes nothing |
| [[state-notes\|notes.md]] | **Hand-written.** Gotchas where the error points away from the cause |

## Not recorded, deliberately

Snapshot membership, nftables counters, Tailscale-managed chains, USB device
numbers. All churn without meaning. The collector records what you control, not
what the system manages on its own.

RFC1918 bind addresses in the socket list are stripped to `<LAN>` rather than
mapped, for the same reason: the address is a DHCP lease that differs per
network, so mapping it would put a churning value in `redact.map` permanently.
`0.0.0.0`, `[::]` and `127.0.0.1` stay literal — which of those a service chose
is the finding. Tailscale CGNAT addresses are still mapped, not stripped.

## These files cannot answer "is `<package>` installed"

Not a deliberate omission like the section above — a **coverage gap**, recorded
here because the natural reading of the two files is that they cover each other and
they do not.

`packages.md` runs `pacman -Qqe`: explicit installs only, so a dependency has never
appeared in it. `services.md` reports *enabled* units and a loaded-session graph, so
it misses anything D-Bus-activated, anything with no unit, and anything running
inside another process. **Both gaps apply at once.** Worked case, 2026-08-28:
`ksecretd`, `klipper` and `ksshaskpass` are on the machine and appear in no section
of either file.

This was harmless while the interesting packages were installed explicitly. Since
`plasma-meta` replaced ~10 explicit desktop packages with one line, it is not.
**Absence from `packages.md` is not evidence of absence from the machine** — run
`pacman -Qi <name>` on the host. See
[[../backlog/collector-desktop-blind-spots|collector-desktop-blind-spots]], which
also covers the battery charge thresholds, not collected at all.

**Randomised MACs are replaced structurally, for the same reason, and the
precedent is the paragraph above.** NetworkManager sets
`wifi.cloned-mac-address=stable`, so the wifi MAC changes per association. The
map is built from the *permanent* address — `gen-redact-map.sh` reads
`ip -d link show`'s `permaddr` — so the rotating address in `## interfaces` never
matches an entry, and on 2026-08-26 a literal `wlp3s0` MAC reached
`state/network.md` with `redact.sh` warning and exiting 0.

Adding the current value to `redact.map` would fix that run and nothing after it:
the next association produces a different address, and the map accumulates a
permanent entry per association. That is exactly the churn the DHCP paragraph
rejects.

So the collector replaces any **locally-administered** address with
`<MAC_RANDOMISED_<iface>>` before redaction sees it. The bit is bit 1 of the
first octet, so the test is whether the second hex digit is one of `2 3 6 7 a b
e f` — no map entry, deterministic, stable across associations. Permanent
addresses are untouched and still go through `redact.map`, which is where they
belong: they identify the hardware, and they do not rotate.

Two consequences worth stating:

- **`## permanent MACs` is where the real address lives**, tokenised as
  `<PERMADDR_wlp3s0>`. Measured 2026-08-26: that field is populated for `wlp3s0`
  and **empty** for `enp2s0f0`, `enp4s0` and `tailscale0` — the kernel reports a
  permanent address separately only when the current one differs from it. So the
  empty rows are the tell that those interfaces are *not* randomised right now.
- **Wired interfaces are not permanently exempt.** `ethernet.cloned-mac-address`
  is `stable` too. Both wired NICs are DOWN, so they currently show permanent
  addresses; when one carries a connection it will be rewritten the same way.

## `redact.sh` warns where the pre-commit hook blocks, and covers more

Recorded because the two look like one mechanism and are not, and because the
gap runs in the unsafe direction.

**Severity differs, and that part is deliberate.** `redact.sh` ends its warning
pipeline with `|| true`, so an unmapped secret-shaped value prints to stderr and
the collector still exits 0 and still writes its output. The pre-commit hook
runs the same kind of scan over staged content and `exit 1`s. That layering is
defensible: the collector is a producer and must finish writing — aborting
mid-run is the stale-`state/` failure
[[../backlog/collector-stale-file-detection|recorded separately]] — while the
commit is the last point at which a human is present. Producer warns, gate
blocks.

**Coverage differed, and that part had no argument behind it — fixed
2026-08-26.** The two pattern sets were otherwise identical, but `redact.sh`
carried a fifth expression the hook did not: `\b[0-9a-f]{32}\b`, machine-id
shaped. A bare `<MACHINE_ID>` value would have been warned about by the collector
and **not blocked** by the gate — the weaker detector having the final say before
a push to a remote, which is backwards. The hook now carries all five.

**The hook's remediation message was wrong for rotating values — fixed the same
day.** It said *"Add to redact.map, or override with `git commit --no-verify`."*
For a value that rotates, both are wrong: the first accumulates a permanent map
entry per association, the second commits the leak. It now offers three options
keyed on whether the value rotates, and points at the MAC rewrite above as the
worked case for the third.

**The hook is tracked now.** It lived at `.git/hooks/pre-commit`, which is not
part of the repository — so the gate existed on one machine, could not be read
from the vault, and was not reviewable, while this file described it as part of
the system. It is at `.githooks/pre-commit` at the repository root, with
`core.hooksPath` pointed at it. Root rather than `arch-thinkpad/collect/`,
because the control is repo-scope: it guards every device folder.

**Two residuals, in [[../backlog/git-hooks-activation-and-parity|the backlog]].**
`core.hooksPath` is still local config, so a fresh clone has the file and no gate
until someone sets it — visible and inert, which beats invisible and absent but
is not automatic. And nothing enforces that the two pattern sets stay identical;
they match today because both were edited in one sitting, which is not a
mechanism.

## Read `harness.md`'s `# source:` markers before its rows

Every section there begins with a `# source:` line saying whether the key it reads was
present in `.claude.json`. That is not decoration and it is the same hazard as the
`luks tokens` one below: **an empty object and a `jq` filter naming a field that has
moved produce identical output — nothing.**

It has already fired once. The first draft of the collector read a top-level
`.mcpServers`; there is no such key, MCP configuration is per-project
(*measured 2026-08-26, `jq keys`*). Every run would have reported an empty MCP list for
a machine that simply stores it elsewhere. A section whose job is to make tool-granting
surface visible is the worst possible place for that failure, so the marker exists to
separate *"none configured"* from *"the collector is looking in the wrong place."*

`ABSENT` means the collector is wrong until proven otherwise. Empty rows under a
`present` marker mean there is genuinely nothing.

**The section is a bounded allowlist, not a dump.** `.claude.json` also holds
`oauthAccount`, `userID` and `machineID`; none is read. A redaction pass would have to
know every field that might carry an identifier in a file this system does not control
and whose schema can gain one at any release — it fails open, and an allowlist does not.

## Read `luks tokens` with care

The jq filter names its fields. A field name that does not match what the
running systemd writes emits `null` — and `"tpm2-pcrs": null` is visually
identical to an enrollment bound to no PCRs at all, which is the failure the
section exists to detect. `systemd-cryptenroll(1)` makes that the default:
binding to no PCRs is "also the default, if this option is not used."

The token schema is not documented in any man page, so the names were taken
from the enrollment this machine actually has rather than looked up. Treat a
`null` as "the collector is wrong" until a non-null run has been seen at least
once, not as "the enrollment is unbound."

## Known-unstable

USB ordering in hardware.md was stabilised 2026-08-21. If any file diffs on a
run where nothing changed, that module needs the same treatment.
