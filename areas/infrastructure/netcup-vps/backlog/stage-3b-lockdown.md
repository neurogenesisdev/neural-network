---
type: backlog-item
status: done
priority: high
device: netcup-vps
projects: [netcup-build]
tags: [backlog, security, nftables, tailscale, ssh, lockout, done]
created: 2026-08-30
updated: 2026-08-31
completed: 2026-08-31
commit: pending
---

# stage-3b-lockdown

> [!NOTE] Summary
> **Done 2026-08-31.** The machine joined the tailnet, tailnet SSH was verified
> before anything was tightened, nftables went to `policy drop` with `tailscale0`
> as the only accepted path, and public inbound was confirmed dead from a fresh
> connection. Steps 1–5 completed; **step 6 is deliberately not done** and is now
> [[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]].
> Step 7 remains open as [[panel-firewall-outer-ring]].

## Outcome

`measured` 2026-08-31.

| Step | State |
|---|---|
| 1 — join tailnet | Done. `tailscale up --accept-dns=false` |
| 2 — record address | Done. `<TS_IPV4>` |
| 3 — the gate | Passed, after the expected `Permission denied (publickey)` |
| 4 — do not proceed until 3 succeeds | Held |
| 5 — tighten nftables | Done. Input policy `drop`, port 22 blanket accept removed |
| 6 — bind sshd to the Tailscale address | **Deliberately not done** — ADR-0029 |
| 7 — panel firewall | Open — [[panel-firewall-outer-ring]] |

Verified across a deliberate reboot: `policy drop` persists, `tailscaled` and
`nftables` both active, tailnet SSH works, public SSH times out.

## What the item predicted correctly

**The ordering was the item, and it held.** Nothing was tightened before tailnet
SSH had returned a prompt on a connection that did not depend on the public path.

**Steps 5 and 6 were not done together**, and in the end 6 was not done at all.

**`nft -c -f` validates syntax, not reachability**, so the scheduled revert was
armed at step 5. It was needed — not because the ruleset was wrong, but because
the *verification* was, and the five-minute timer expired while a false negative
was being chased. The machine reverted to permissive on its own and nothing was
lost. A revert that fires during a real session is the mechanism working, not a
mistake.

## What the item did not anticipate

**`flush ruleset` in Debian's stock `/etc/nftables.conf` wipes Tailscale's own
tables.** Tailscale manages `table ip filter`, `ip6 filter`, and the `nat` and
`mangle` tables through `iptables-nft`, each marked *"do not touch"*. A revert
file built by copying the stock config would have torn out the tailnet's firewall
rules at the exact moment the tailnet was the only way in — `tailscaled` rebuilds
them eventually, which is not a property to depend on during a recovery.

Both `/etc/nftables.conf` and `/etc/nftables.conf.permissive` are now scoped:

```nft
table inet filter
delete table inet filter

table inet filter { ... }
```

The bare declaration before the `delete` is load-bearing — without it, `delete`
on an absent table aborts the whole file, which in the revert file would mean a
recovery that fails on the one path it exists for.

**SSH connection multiplexing invalidated the verification.** Covered in
[[../sessions/20260831-stage-3b-lockdown#The verification that lied|the session
log]]; the short form is that `ssh netcup-public` returned a prompt *after* the
firewall was closed, because it rode a master connection opened before the
change. A `policy drop` ruleset filters new connections and says nothing about
existing ones. Any check of a firewall change over SSH must force
`-o ControlPath=none` or it is measuring the past.

## Files this produced

- `/etc/nftables.conf` — tightened, scoped, `policy drop`
- `/etc/nftables.conf.permissive` — the revert target, scoped
- `/etc/nftables.conf.bak` — the pre-change stock file, kept
- `~/.ssh/config` on arch-thinkpad — `netcup-public` and `netcup-tailnet` blocks,
  both **above** `Host *`, with `ControlPath none` on the tailnet block

## One loose end this created

[stage3-baseline.sh](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage3-baseline.sh) carries a postcondition
asserting the input policy is still `accept` — a guard against someone tightening
the firewall without passing the gate. That gate has now been passed, so
**re-running stage 3 on this machine would fail that check**, correctly but
confusingly, and re-running the script as a whole would reinstate the permissive
ruleset.

Not fixed here because the right fix depends on what `build/` is for: if it
describes *building a netcup host from bare metal*, the postcondition is right and
the script should simply never be re-run against a built machine. If it describes
*this machine's current configuration*, it is now wrong. That question is open and
belongs with [[netcup-collector]], which is where re-verification would live.

## Related

- [[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]] — step 6, closed as won't-do
- [[panel-firewall-outer-ring]] — step 7, now unblocked
- [[../sessions/20260831-stage-3b-lockdown|session log]]
- [[../runbooks/ssh-lockout-recovery|Locked out of SSH]]
- [stage3-baseline.sh](https://github.com/neurogenesisdev/infra-tools/blob/main/netcup-vps/archive/debian-2026-08/stage3-baseline.sh) — the permissive baseline this replaced
