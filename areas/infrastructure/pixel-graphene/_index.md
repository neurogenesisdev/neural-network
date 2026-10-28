---
type: index
device: pixel-graphene
updated: 2026-08-31
tags: [index, stub]
---

# pixel-graphene

Google Pixel 9 Pro XL running GrapheneOS. Tailnet member.

> [!WARNING] This folder is a stub, created 2026-08-31.
> It exists because the device acquired its first documented configuration this
> day — SSH over the tailnet, in both directions. Everything below is `measured`
> or `stated` on that date and **nothing re-verifies it**; there is no collector
> for this device and one is not planned. Treat it as a starting point, not as a
> description of the phone.

## What is recorded here so far

### Tailnet

Joined, address `<TS_IPV4>`. Reachable from arch-thinkpad; reaches arch-thinkpad
and netcup-vps.

> [!IMPORTANT] Node key expiry was set and is the first thing to check.
> On 2026-08-31 the Tailscale admin panel showed this device as *"Expires 17
> hours from now"* — the only one of the four machines with an expiry pending.
> When a node key lapses the device drops off the tailnet and **every route
> described on this page stops working**, including the SSH access set up the
> same day. Whether expiry has since been disabled in the admin panel is
> **not verified**.

### SSH

Termux provides both the client and the server.

| Direction | Path | Notes |
|---|---|---|
| arch-thinkpad → pixel-graphene | `Host pixel-tailnet` on the laptop | **Port 8022**, not 22 — Termux's sshd default. User is Termux's `u0_a…` account |
| pixel-graphene → arch-thinkpad | `Host thinkpad` in Termux's `~/.ssh/config` | Uses `~/.ssh/id_ed25519_pixel`, generated on the phone. Depends on the laptop's sshd, enabled the same day — see [[../arch-thinkpad/backlog/undeliberate-listeners-review\|undeliberate-listeners-review]] |
| pixel-graphene → netcup-vps | Termux, key already authorised | Worked before either of the above; it is what proved the phone's client side was fine while arch-thinkpad's sshd was still `inactive` |

**The phone's key is on disk, not in an agent.** Unlike arch-thinkpad, where the
private half lives in KeePassXC's agent and `IdentityFile` names the `.pub`,
Termux holds a plain private key file. That is a different trust tier and is not
yet recorded anywhere as such.

**Termux's `sshd` does not survive a reboot or aggressive dozing.** It needs
`termux-boot` plus a wake lock to persist, and GrapheneOS's battery management
will still stop it. Treat phone-side sshd as started on demand, not as a service.
`termux-boot` is **not installed** as of 2026-08-31.

## Trust position

pixel-graphene and arch-thinkpad are **peers** — both endpoints under physical
control, both holding private keys.
[[../netcup-vps/decisions/20260830-vps-holds-no-credentials-upward|ADR-0027]]
governs the VPS reaching upward and does not constrain this pair.

That record is expected to move to `shared/`, at which point this page should
link the moved copy.

## What is not recorded here

Everything else about the device: GrapheneOS configuration, profiles, app
sandboxing, Mullvad and exit-node behaviour, Syncthing (which carries the
KeePassXC database to this phone and is currently the reason the database has an
off-machine copy at all). None of it is written down, on this page or elsewhere.

Filling that in is not currently a backlog item on any device. If it becomes one,
it belongs here rather than under arch-thinkpad.
