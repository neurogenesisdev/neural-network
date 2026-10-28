---
title: "netcup panel firewall: configure the outer ring"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-09-11
commit: pending
created: 2026-09-11
updated: 2026-09-11
tags: [runbook, firewall, netcup, tailscale, security]
---

# netcup panel firewall — configure the outer ring

> [!NOTE] Summary
> Plan step 0.3, closing [[../backlog/panel-firewall-outer-ring|the backlog item]].
> The cloud firewall in the SCP is the one control that survives a compromised
> host. Today it accepts everything inbound. Target: inbound allows only
> Tailscale's UDP port and ICMP; default inbound drop; outbound untouched
> (the SMTP block stays). One change at a time, verify between, keep a
> tailnet SSH session open throughout. ≈ 1 h including verification.

## Executed 2026-09-11 — what the UI actually does

- The per-server firewall is **server → Firewall**, not the account-level
  "Firewall policies" page (which may read *No firewall policies defined*).
- Policies are objects; create one, then **drag it into order** on the
  server's firewall page. Defaults can be restored with *Restore Default Policies*.
- **Empty address/port fields mean "any".** The UI renders that as `*:*`, but
  typing `*:*` fails validation. The port goes in **Dst Ports**, not *Destinations*.
- No `ANY` protocol — one DROP rule each for TCP and UDP. ICMP is matched by
  the *Ping allow* policy above, so it needs no rule here.
- The panel **validates before applying**, so a malformed rule cannot lock you
  out; a well-formed wrong one still can.
- Applied set: policy `tailnet-only` = `INCOMING UDP ACCEPT dst 41641`,
  `INCOMING TCP DROP`, `INCOMING UDP DROP`. Verification results in
  [[../sessions/20260911-panel-firewall-outer-ring|the session log]].

## Preconditions (measured before you start)

- [[../backlog/stage-3b-lockdown|stage-3b-lockdown]] done 2026-08-31: nftables
  `input policy drop`, reachable over `tailscale0`, verified across a reboot.
- Tailscale path has been working for days — it has.
- Two independent sessions open before any change: a tailnet SSH session
  **and** the SCP Screen (VNC) tab. The VNC console is the path that does not
  depend on the firewall.

## Order of operations

The panel evaluates rules top-to-bottom, first match wins, and the **Implicit
Rules** block at the end currently accepts all inbound. Changes apply
immediately, no dry run, no scheduled revert (backlog item, measured
2026-08-30).

1. **Record the current state** — screenshot the rule list into the session
   log (no addresses need redaction here; there are none in the rules).
2. **Add explicit inbound accepts, above the implicit block:**
   - UDP `41641` from any → this host (Tailscale direct connections).
     Confirm the port with `tailscale debug prefs | grep -i port` or
     `ss -ulpn | grep tailscaled` on the host — if you set a fixed port it is
     that one; the default is 41641 (`inferred`; the docs say so).
   - ICMP / ICMPv6 — already present as "netcup Ping allow"; leave it.
3. **Verify nothing changed yet:** the SSH session still works; from the phone
   on mobile data `nc -zvw3 <public IPv4> 22` still *connects* (the host's
   nftables drops it silently, so expect a timeout, not a refusal — that is
   the inner ring working; the outer ring is not yet active).
4. **Add the final inbound deny** (or flip the implicit inbound policy to
   drop, whichever the panel offers). Do not touch outbound.
5. **Verify, in this order:**
   - Existing tailnet SSH session still alive (it is `established`; the panel
     firewall may or may not track state — this is what you are finding out).
   - **Fresh** tailnet SSH session from the laptop: `ssh -o ControlPath=none
     netcup-tailnet true && echo NEW-SESSION-OK` — bypass the multiplexer,
     otherwise you are testing the old connection (lockout runbook §0).
   - `tailscale status` on the host: the laptop peer shows a **direct**
     connection, not `relay "fra"`. If it shows relay, UDP 41641 is not
     getting through — fix the accept rule before continuing. Degraded-but-
     working is the failure mode the backlog item warns about.
   - From the phone on mobile data: `nc -zvw3 <public IPv4> 22` → timeout
     (unchanged from step 3, but now for two reasons).
   - `ping <public IPv4>` from the phone → replies (ICMP allow intact).
6. **Reboot test** (the lockdown was verified across a reboot; this must be
   too): `sudo systemctl reboot`, wait, fresh SSH over the tailnet, `tailscale
   status` direct again.

▶ If step 5 fails for the fresh session: the VNC console is open; remove the
deny rule in the panel; the session returns. That is the whole rollback.

## Record

- Update `backlog/panel-firewall-outer-ring.md`: status resolved, date, the
  observed answer to "does the panel firewall track state" (whether the old
  session survived step 4).
- Add a row to `_index.md`'s status paragraph: outer ring configured.
- The rule set belongs in `state/` — which does not exist — so it goes into
  the session log for today until the collector exists
  ([[../backlog/netcup-collector|netcup-collector]]).

## Not in scope

- Outbound rules. The SMTP block stays; a future mail-sending service goes
  through a relay on 587 via a provider that the block does not cover, or the
  block is lifted deliberately then.
- Anything about sshd's bind address — [[../decisions/20260831-sshd-binds-wildcard-nftables-is-the-control|ADR-0029]].

## Related

- [[../backlog/panel-firewall-outer-ring|backlog item]] · [[ssh-lockout-recovery|Locked out of SSH]] · [[../sessions/20260831-stage-3b-lockdown|stage-3b session log]]
