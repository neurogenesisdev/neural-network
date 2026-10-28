---
title: "Locked out of SSH"
type: runbook
provenance: measured
device: netcup-vps
stale: false
last_verified: 2026-08-31
commit: pending
created: 2026-08-30
updated: 2026-08-31
tags: [runbook, ssh, recovery, console, vnc, known-hosts, multiplexing, terminfo]
---

# Locked out of SSH

Four faults present as "I cannot SSH in" and the repairs differ. **Diagnose
before acting** — three of the four are client-side and the machine is fine.

> [!WARNING] Read section 0 first if anything about SSH changed recently.
> A connection that still works can be an *old* connection. The check below takes
> five seconds and prevents a whole class of wrong conclusion.

> **Before anything else: check the SCP Screen tab.** It answers "is the machine
> up, and at what stage" in one look, and three of the four faults below are
> distinguishable from that screen alone. Retrying `ssh` tells you nothing new.

## 0a — `Permission denied (publickey)` for every user, after time away

Run `ssh -v <host> true 2>&1 | grep identities`. If it says `agent contains no
identities`, KeePassXC has locked (its inactivity timer — ADR-0004) and the
agent holds no key. Unlock the database; `ssh-add -l` should list the key.
The server is fine. Measured 2026-09-11 after a shift and a hotspot change.

## 0 — Is this connection actually new?

**Applies whenever a shell, key, sshd config or firewall rule has just changed.**
This is not a lockout fault; it is the fault that makes you *misdiagnose* one.

`Host *` in `~/.ssh/config` sets `ControlMaster auto` with `ControlPersist 10m`.
Every `ssh` to a host with a live master is a channel inside the existing
connection, not a new login. So:

- **A change to the login shell will not appear.** The master keeps serving the
  shell it was born with.
- **A firewall that now drops inbound will still let you in.** `policy drop`
  filters *new* connections; `ct state established,related accept` keeps the
  master alive, correctly.

The tell is on every disconnect:

```
Shared connection to <SERVER_ID>.ultrasrv.de closed.
```

That line means multiplexed. Before trusting any post-change check:

```bash
# arch-thinkpad
ls -1 ~/.ssh/sockets/          # every file here is a live master
ssh -O check <host>            # "Master running (pid=…)" or no such file
ssh -O exit <host>             # tear it down
```

Or force the check itself to be independent — command-line options beat the
config, so this works even when the config is wrong:

```bash
ssh -o ControlPath=none -o ConnectTimeout=10 <host>
```

**This produced a false confirmation on 2026-08-31**: `ssh` to the public
hostname returned a prompt after stage 3b had correctly closed the public path.
Acting on that reading — assuming the ruleset was broken — would have meant
editing a firewall that was already right. See
[[../sessions/20260831-stage-3b-lockdown#The verification that lied|the session
log]].

`netcup-tailnet` carries `ControlPath none` in its host block for this reason: the
recovery route must never be a channel inside a connection that a change can
kill.

### The same mechanism, twice, in the same file

**`ssh_config` takes the FIRST value it obtains for each option**, so a `Host *`
block at the top overrides every block below it. This has now cost a round twice:
`IdentitiesOnly` (fault A below, 2026-08-30) and `ControlPath` (2026-08-31).

**`Host *` belongs at the bottom of `~/.ssh/config`.** Per-host blocks above it,
shared defaults below, where they fill in only what is unset.

```bash
# arch-thinkpad — what is actually in effect, config parsing included
ssh -G <host> | grep -Ei 'controlpath|identityfile|user|hostname'
```

## Which fault

| Symptom | Fault | Machine is |
|---|---|---|
| `Permission denied (publickey)` | **A** — client offers no key | Fine |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | **B** — host key changed | Fine, or in rescue |
| Timeout, no banner | **C** — network, firewall, or not booted | Unknown — go to the console |
| `Connection refused` | **D** — reachable, sshd not running | Booted, sshd down |

## A — `Permission denied (publickey)`

**This has already happened once, on first contact after the rebuild**, and the
cause was client-side.

```bash
# arch-thinkpad
ssh -v luiswg@<SERVER_ID>.ultrasrv.de 2>&1 \
  | command grep -E 'Offering|Will attempt|identity file'
```

If there is **no `Offering public key:` line**, the client never sent a key and
the server is blameless. The cause is `IdentitiesOnly yes` under `Host *` in
`~/.ssh/config`: it restricts offers to keys named by `IdentityFile`, and with
none specified it falls back to default filenames that do not exist. The
agent-held key is filtered out despite being loaded.

Confirm the mechanism without changing anything:

```bash
ssh -o IdentitiesOnly=no luiswg@<SERVER_ID>.ultrasrv.de
```

Fix — a host block **above** any `Host *` section, since SSH takes the first
value it sees for each option. Current blocks as of 2026-08-31, both above
`Host *`:

```
Host netcup-public
    HostName <SERVER_ID>.ultrasrv.de
    User luiswg
    IdentityFile ~/.ssh/id_ed25519_t14.pub

Host netcup-tailnet
    HostName <TS_IPV4>
    User luiswg
    IdentityFile ~/.ssh/id_ed25519_t14.pub
    ControlPath none
```

**`netcup-public` no longer reaches the machine** — stage 3b closed public
inbound on 2026-08-31, so it times out by design. It is kept because a *timeout
there plus a working `netcup-tailnet`* is the signature of a correctly locked-down
host, and the reverse pair is the signature of a firewall that failed to load.

**`IdentityFile` points at the `.pub` file, not the private key.** The private
half lives in KeePassXC's agent and is not on disk; naming the public key tells
SSH which agent identity to offer.

If a key *is* offered and still rejected, the authorised set is wrong — go to the
console and read `/home/luiswg/.ssh/authorized_keys`. Two keys should be present,
commented `t14-202608` and `pixel-graphene`.

## B — host key changed

Legitimate in exactly two situations: the system was rebuilt, or the machine is
running the **rescue system**, which is a different OS with its own host keys.
Indistinguishable from an attack without verification, so verify.

Current host key fingerprints, generated 2026-08-30 during stage 2 and read from
the target filesystem before it ever booted. **Not redacted — a host key
fingerprint is offered to every client that connects and is public by
construction.**

| Algorithm | Fingerprint |
|---|---|
| ED25519 | `SHA256:5yPvvUali86s9ILHHqhzf/L4CtZONsISQURsN0L7rMo` |
| ECDSA | `SHA256:YBBbmBK09x0pN0G4/exP1BFr4ylWAeavIT6Ez5H9/QI` |
| RSA | `SHA256:zKjgJ9P32/7GE3JB0dYq2S4AfB2Oqyer6rUw0elKPE8` |

Modern clients negotiate ED25519; that is the row that will normally be offered.

If the offered fingerprint **matches**:

```bash
# arch-thinkpad
ssh-keygen -R <SERVER_ID>.ultrasrv.de
ssh-keygen -R <NETCUP_IPV4>
```

If it **does not match**, and the machine is not in rescue and has not been
rebuilt: stop and investigate. Do not clear the entry.

If the machine *is* in rescue, the mismatch is expected — rescue keys are
generated per activation and are not recorded anywhere. Verify against the
fingerprint shown in the SCP, or read it from the rescue console:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

## C — timeout

The machine is not answering. Open the SCP **Screen** tab.

| Console shows | Meaning | Do |
|---|---|---|
| Login prompt | Booted, sshd not reachable | Log in as `luiswg`, check `systemctl status ssh` and `nft list ruleset` |
| GRUB menu or kernel panic | Boot failure | [[rescue-system-entry\|Enter rescue]] |
| Nothing, black | Powered off or hung | SCP → Control → power state |
| Emergency shell | fstab or filesystem fault | See below |

**The most likely self-inflicted cause is the firewall.** The stage-3 baseline is
deliberately `policy accept`; if it has since been tightened, verify the tailnet
path works before assuming the public path should be closed. That ordering is the
whole point of [[../backlog/stage-3b-lockdown|stage-3b-lockdown]].

**The second most likely is the interface name.** `net.ifnames=0` must be present
in `GRUB_CMDLINE_LINUX` or the NIC comes up as `ens3`, the `eth0` stanza in
`/etc/network/interfaces` matches nothing, and the machine boots with no network.
From the console:

```bash
cat /proc/cmdline | grep -o 'net.ifnames=0' || echo "MISSING — this is the fault"
ip -br addr
```

**An emergency shell after an fstab edit** is the third. Six btrfs mounts and one
ext4; a wrong `subvol=` or a stale UUID drops the boot into emergency. The
console can still log in as `luiswg`; `journalctl -xb` names the failing unit.

## D — connection refused

Reachable, nothing listening on 22. Console in, then:

```bash
systemctl status ssh
journalctl -u ssh -b --no-pager | tail -30
sudo sshd -t          # config syntax check — silent means valid
```

`sshd -t` is the one that matters after an `sshd_config.d/` edit: a syntax error
makes the daemon refuse to start, and the failure appears at the next restart
rather than at edit time.

## Console access, in full

SCP → Screen. Log in as **`luiswg`** with the console password from KeePassXC,
then `sudo -i`. Root cannot log in — see
[[areas/infrastructure/netcup-vps/runbooks/_index#The recovery path this machine actually has|the index]] for why, and why
the panel's root-password reset is not the answer.

**The console keyboard layout is `en-us` and the operator's keyboard is German.**
`y`/`z` are transposed and most symbols are elsewhere. This matters most when
typing a password, where there is no echo to reveal the problem. The layout is
changeable in SCP → Settings → Keyboard Layout; changing it before an emergency
is better than discovering it during one.

## Not a lockout: the terminal looks broken over SSH

Symptoms: `WARNING: terminal is not fully functional`, `tput: unknown terminal
"xterm-kitty"`, pagers refusing to page, multi-line pastes rendering as one
scrolling line, history truncated at the first newline.

The machine and the shell are both fine. kitty announces `TERM=xterm-kitty`, SSH
passes that name to the remote, and a host with no terminfo entry for it falls
back to dumb-terminal behaviour — readline cannot redraw a multi-line buffer.

```bash
# NetCup server
sudo apt install -y kitty-terminfo     # NOT ncurses-term, which does not carry it
tput lines                             # a number means fixed
```

Host-independent form, no root needed on the far end:

```bash
# arch-thinkpad
infocmp -x xterm-kitty | ssh <host> 'tic -x -o ~/.terminfo /dev/stdin'
```

**This is a standing property of kitty, not of any one host** — it also hit the
Grml rescue environment on 2026-08-30. `kitten ssh <host>` copies terminfo
automatically on every connect and is the durable fix; not yet adopted.

Two shell notes that travel with it, since they surface at the same moment:

- **zsh runs `#` as a command** in interactive shells by default, so every
  platform-labelled code block in this vault (`# NetCup server`) errors on a
  fresh zsh. `setopt INTERACTIVE_COMMENTS`.
- **bash leaves the pasted-text highlight on screen after `^C`**. `set
  enable-active-region off` in `~/.inputrc`.

## Escalation

If the console cannot log in, or the system does not reach a login prompt, the
next step is [[rescue-system-entry|the rescue system]] — a separate boot
environment with its own credentials, from which the disk can be mounted and
repaired offline.
