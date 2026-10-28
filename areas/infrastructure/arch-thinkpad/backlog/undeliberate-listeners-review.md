---
type: backlog-item
status: resolved
priority: medium
device: arch-thinkpad
projects: [security]
tags: [listeners, network, hardening, fwupd, avahi, ssh, tailscale]
created: 2026-08-21T00:00:00Z
updated: 2026-09-02T00:00:00Z
commit: pending
---

# Review the three undeliberate listeners

> [!NOTE] Summary
> `ollama` (127.0.0.1:11434), `passimd` (0.0.0.0:27500) and `avahi-daemon` (0.0.0.0:5353) all listen without having been deliberately enabled. None is reachable — the nftables input chain is default-drop — but two bind all interfaces and none appears in any document.
> **A fourth listener now exists and is deliberate** — `sshd`, enabled 2026-08-31
> and bound to the tailnet address. Recorded below so it is not rediscovered as a
> finding.

> [!INFO] Re-measured 2026-09-02, no decision taken
> `ss -tulnp`: `avahi-daemon` on `0.0.0.0:5353`/`[::]:5353`, `passimd` on
> `0.0.0.0:27500`, `ollama` on `127.0.0.1:11434`, `sshd` on `<TS_IPV4>:22`.
> Enable states: `avahi-daemon.service` **disabled** and `avahi-daemon.socket`
> disabled — yet it is running, so something activates it on demand;
> `passim.service` is `static` and `passim` is a hard dependency of `fwupd`
> (`Depends On`), which is the likely activator of both; `ollama.service` is
> **enabled** by hand or by package. `avahi` is `Required By` `libcups`,
> `ostree`, `passim`, `phodav`, `pipewire-pulse`, `tinysparql` — so it cannot be
> removed, only left unstarted. The cheapest closing move is masking `passim` and
> `avahi-daemon` and switching `ollama` to socket activation; each is a choice
> with a rationale, so the ADR this item asks for is still the right home.

> [!SUCCESS] Resolved 2026-09-02 by [[../decisions/20260902-what-listens-and-why|ADR-0031]].

## Findings

| Listener | Bind | Origin | Question |
|---|---|---|---|
| `ollama` | loopback | Installed and enabled at boot | Is it wanted at boot, or on demand? |
| `passimd` | all interfaces | Ships with `fwupd`; advertises firmware metadata over mDNS | Disable? |
| `avahi-daemon` | all interfaces | mDNS service discovery | Anything depend on it? |

## Deliberate listeners — not findings

`measured` 2026-08-31.

| Listener | Bind | Why it is here |
|---|---|---|
| `sshd` | `<TS_IPV4>:22` | Enabled 2026-08-31 so pixel-graphene can reach this machine over the tailnet. Bound to the tailnet address in `/etc/ssh/sshd_config.d/20-tailnet-only.conf`, with `PasswordAuthentication no` |
| `syncthing` | `*:22000` | Pre-existing and intended — it is what carries the KeePassXC database to pixel-graphene |

**`sshd` was `inactive` until 2026-08-31**, which is worth stating because it
inverts an assumption: the laptop was reachable *from* nowhere, and a phone-side
client failure looked like a key problem for a round before `systemctl is-active`
was checked.

The bind is deliberate and the reasoning is the mirror of
[[../../netcup-vps/decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]],
which rejected `ListenAddress` on netcup-vps. There, sshd failing to bind because
`tailscaled` had not come up yet would strand the operator behind a VNC console;
here the machine is in his hands and the same failure costs a reboot. **Physical
access flips which failure mode is acceptable**, so the two machines take opposite
decisions from the same argument. If sshd ever fails to start after a boot, that
is the race, and `After=`/`Wants=tailscaled.service` on the unit is the fix.

**The race fired on the first reboot, 2026-09-01, and is closed 2026-09-02**
(`measured`). sshd was `failed (start-limit-hit)` for a day: all five retries
completed 90 ms before `tailscale0` existed. The fix is the drop-in named above,
plus two things the sentence above did not anticipate — `After=tailscaled.service`
alone is not sufficient because `tailscaled` signals ready before the interface
carries an address, so the drop-in also polls for the address in `ExecStartPre=`,
and the default start-rate limit is disabled so a slow tailnet produces
`activating (auto-restart)` rather than a permanent `failed`. Diagnostic detail in
[[areas/infrastructure/arch-thinkpad/state-notes#sshd-start-limit-hit-after-boot-is-the-tailnet-bind-racing-tailscaled|notes]].
The trade-off stands as argued: sshd now depends on tailscaled at boot, and the
price of that dependency failing is a visible stuck unit on a machine in the
operator's hands, not a stranded server.

This has not been written up as an ADR. It probably should be, alongside whatever
is decided about the three listeners above — one record covering *what listens on
this machine and why* is better than four scattered notes.

## Note

The working assumption "nothing listens that I did not put there" is false. Whatever is decided, the outcome belongs in a decision record — this is a hardening choice with a rationale, not a gotcha.

Check whether anything depends on avahi before disabling; some printing and file-sharing paths use it.

**Roaming is the property that makes the bind matter.** Unlike netcup-vps, this
machine joins untrusted networks. A listener on `0.0.0.0` is offered to every
other client on a café or hotel LAN; the nftables default-drop is what currently
makes that harmless for the three above, which means **that ruleset is
load-bearing on a machine that changes networks daily** and nothing re-verifies
it.
