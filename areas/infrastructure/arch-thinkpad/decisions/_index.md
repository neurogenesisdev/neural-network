---
type: index
device: arch-thinkpad
updated: 2026-09-02
tags: [index]
---

# Decisions

Architecture Decision Records for arch-thinkpad. Append-only — to reverse a
decision, write a new record and set `superseded-by` on the old one rather than
editing it. The history of a reversed decision is why the next person does not
reverse it back.

## Decision Index

| ADR | Status | Impact | Date | Summary |
|-----|--------|--------|------|---------|
| [[20260817-rootless-podman-over-docker\|ADR-0001]] | accepted | medium | 2026-08-17 | Rootless podman writes zero nftables rules — measured, identical `inet filter` hash before and after. Docker would have fought the hand-written default-drop chain |
| [[20260816-hyprland-ipc-never-granted\|ADR-0002]] | **proposed** | high | 2026-08-16 | `hypr/**` is never granted in an enforced profile — the IPC socket accepts `dispatch exec`, so any profile permitting it is decorative. Costs confinement of the compositor shell, not of everything |
| [[20260818-devcontainer-agent-exposure\|ADR-0003]] | accepted | high | 2026-08-18 | Dev Containers forwards `SSH_AUTH_SOCK` unconditionally and cannot be stopped (vscode-remote-release#11413). Accepted; the exposure is *use* of the key, not disclosure, and is bounded by the KeePassXC timer |
| [[20260819-noctalia-userland-consolidation\|ADR-0004]] | **superseded** by [[20260828-plasma-replaces-hyprland-userland\|ADR-0018]] | medium | 2026-08-19 | Ten userland packages replaced by one beta. Superseded 2026-08-28 **by events, not by reasoning** — the trade was never re-argued, its subject stopped existing. Three consequences outlived it and were re-homed |
| [[20260813-dns-ownership-by-mode\|ADR-0005]] | accepted | medium | 2026-08-13 | **Rejects PRD §7.3.** NextDNS over DoT sends account-identifying queries outside the tunnel. Resolution belongs to whoever owns the tunnel. Amended 2026-08-21 — exit nodes now work, so the default-mode question is open |
| [[20260816-container-storage-and-bind-mounts\|ADR-0006]] | accepted | high | 2026-08-16 | Container storage is a nested subvolume, invisible to snapper in both directions. Correct for images, wrong for data — so project and agent state are bind-mounted from `@home`, never named volumes |
| [[20260818-base-image-pinned-by-digest\|ADR-0007]] | **superseded** by [[20260825-devbox-pinning-four-layers\|ADR-0016]] | medium | 2026-08-18 | The image is the version pin, not the Containerfile — both agent installers fetch latest at build time. Superseded 2026-08-25: the installers were measured to accept a version, so the evidence changed rather than the reasoning being wrong |
| [[20260818-restic-exclusions\|ADR-0008]] | accepted | medium | 2026-08-18 | Excluding `/home/.snapshots` cut a 73 GiB scan to nothing. Amended 2026-08-21 — it costs off-disk granularity, and the mitigation is restic cadence (ADR-0012), not re-inclusion. Amended 2026-08-25 — its open question is answered by ADR-0017: the agent-state exclusion narrows to credentials and cache, session history is kept. Amended 2026-08-30 — the fail-open residual **fired**, a 12 GiB VM disk image was found in the set, and the list is now grounded on four named grounds of which *size* is not one. **Declares its own defect:** four ADR-0017 credential exclusions were dropped without being re-argued |
| [[20260815-t5-rescue-partition-unencrypted\|ADR-0009]] | accepted | high | 2026-08-15 | The rescue partition is plain ext4 on purpose — disk images are opaque without the LUKS passphrase, and the Arch ISO has no VeraCrypt. Nothing else goes on it |
| [[20260819-zsh-without-plugin-framework\|ADR-0010]] | accepted | medium | 2026-08-19 | No oh-my-zsh. A large auto-sourced plugin ecosystem in an unconfined host shell with read access to `~/.ssh` is the same exposure class as unconfined Electron, with a worse supply chain |
| [[20260820-thunar-retained-during-yazi-transition\|ADR-0011]] | **superseded** by [[20260828-plasma-replaces-hyprland-userland\|ADR-0018]] | low | 2026-08-20 | Thunar reinstalled deliberately as a fallback during the yazi transition. Superseded 2026-08-28 and **its own trigger never fired** — the package left with the desktop switch, not because yazi became fluent. Dolphin is the GUI file manager now |
| [[20260818-backup-cadence-open\|ADR-0012]] | accepted | medium | 2026-08-18 | Two restic snapshots, six days apart, both manual — and the gap spanned real work. **Decided 2026-08-30, option D.** The cadence question *dissolved* rather than being answered: A, B and C were all workarounds for a target that is only sometimes attached, and a reachable remote removes the constraint that made them candidates. Daily timer, `Persistent=true` |
| [[20260823-tpm2-pcr-selection\|ADR-0013]] | accepted | high | 2026-08-23 | Binds PCRs 0+2+7 with a PIN. SPI write protection is `Disabled` and `action-contact-oem`, so firmware modification is reachable from a root shell — PCR 7 alone would not move. Costs re-enrollment on every firmware update |
| [[20260823-continuous-trim-both-layers\|ADR-0014]] | accepted | high | 2026-08-23 | **Rejects PRD §1.2 and §1.4.** Continuous TRIM at dm-crypt and btrfs, accepting the documented metadata leak — the PRD's weekly-`fstrim` alternative discloses the same thing and cannot run without the grant §1.2 refuses. Two layers reached by one cmdline token |
| [[20260825-devbox-base-is-arch\|ADR-0015]] | accepted | high | 2026-08-25 | **Reverses Block-C §2.** The devbox base becomes Arch — measured parity, gawk 5.4.1 and GNU grep 3.12 identical both sides, closing the mawk class rather than mitigating it. Costs the guard rail the distro difference used to be |
| [[20260825-devbox-pinning-four-layers\|ADR-0016]] | accepted | high | 2026-08-25 | **Supersedes ADR-0007.** Four pins — base digest, ALA repo snapshot, agent installer versions, built-image consumption — plus OCI labels as the image's own identity. A frozen snapshot gets no security updates; the mitigation is a *measured* monthly ceiling, not a remembered one |
| [[20260825-harness-state-and-identity\|ADR-0017]] | accepted | high | 2026-08-25 | **Answers ADR-0008's open question.** Stable project identity now, path parity deferred with a condition; four categories of harness data, and only the mechanisms that can actually act on `$STATE/claude`. Session history joins restic; credentials do not |
| [[20260828-plasma-replaces-hyprland-userland\|ADR-0018]] | accepted | medium | 2026-08-28 | **Supersedes ADR-0004 and ADR-0011.** KDE Plasma replaces the Hyprland/Noctalia userland. Exists mainly to carry three consequences that outlived their parent — clipboard history moves to Klipper with the same exposure and the same open `x-kde-passwordManagerHint` question, `upower` still required, power split into ADR-0020 |
| [[20260828-secret-service-provider-is-ksecretd\|ADR-0019]] | accepted | high | 2026-08-28 | **An unannounced change, ratified after the fact.** `ksecretd` holds `org.freedesktop.secrets`; KeePassXC detected another holder and declined, so its setting is greyed out rather than switched off. KWallet won *uncontested*, not by a race. **Conditional** — reopens if KWallet has no per-request retrieval prompt |
| [[20260828-power-management-stays-with-tlp\|ADR-0020]] | accepted | medium | 2026-08-28 | TLP owns power; `power-profiles-daemon` must never be installed. Sharper under Plasma — powerdevil surfaces a profile switcher that silently does nothing and the obvious fix breaks the charge thresholds. Corrects ADR-0004's 75/80 to a measured **85/90** |
| [[20260830-remote-repository-is-b2\|ADR-0028]] | accepted | high | 2026-08-30 | **Unblocks ADR-0012 and closes `remote-restic-repository`.** B2 over the S3 backend, bucket-scoped key, 12.809 GiB stored for ~$0.03/mo. netcup Storage Space rejected — **NFS only**, IP-allowlisted, useless to a roaming laptop; block storage rejected on **correlation**, not on the reachability the web search claimed. The Storage Box recorded as owned turned out not to exist |

| [[20260902-dotfiles-bare-git-repo\|ADR-0030]] | accepted | medium | 2026-09-02 | **Closes `dotfiles-versioning`.** Bare git repo, work tree `$HOME`, explicit allowlist — the allowlist *is* the answer to "which files hold decisions". chezmoi rejected on weight for a one-workstation setup; revisit at the second machine. Private GitHub remote added the same day |

| [[20260902-what-listens-and-why\|ADR-0031]] | accepted | medium | 2026-09-02 | **Closes `undeliberate-listeners-review`.** `passim` and `avahi-daemon` masked, `ollama` disabled and started by hand; `sshd` (tailnet-bound) and `syncthing` are the deliberate listeners. The firewall stays the control; this stops leaning on it |
| [[20260902-plasma-login-path-accepted\|ADR-0032]] | accepted | medium | 2026-09-02 | **Closes `login-path-unreviewed`.** The DM stacks exactly what console login stacks plus `pam_kwallet5`, so wallet and login are one secret — accepted behind FDE. No autologin, `getty@` kept as fallback |

| [[20260903-agent-containment-two-lanes\|ADR-0033]] | accepted | high | 2026-09-03 | Coding in the devbox with tools and the ArchWiki baked into the image; egress reaches the internet **and the tailnet** (n8n on the VPS), not the LAN or the host. Administration on the host as root under the vault's rules. VS Code made as safe as the wrapper by launching it without the agent socket. Ends the add-a-tool-per-session loop; rejects an egress allowlist |

## Open

**One** record is `proposed` rather than `accepted`: **ADR-0002**, whether the
Hyprland IPC rule is worth its cost. ADR-0012 was the other and was decided on
2026-08-30 — after twelve days, and only because the thing blocking it
(a reachable remote) was built, not because the four options were re-weighed.

**ADR-0008 declared a defect in its own latest amendment and then closed it the same
evening** — four ADR-0017 exclusions dropped without being re-argued, `paste-cache`
among them, restored once the operator had read the argument. Not open any more, and
kept on this list for one round because the *sequence* is the point: the record
names how a credentials-ground exclusion was removed by a storage-cost argument, and
that survives the fix. The half that remains — why nothing would have caught it — is
[[../backlog/restic-exclusion-list-maintenance|restic-exclusion-list-maintenance]].

**ADR-0002's premise moved on 2026-08-26 and has not been re-established.** Hyprland
is uninstalled; the equivalent privilege is on the KDE session bus, which AppArmor
mediates differently, and the set the rule leaves unconfined is now the session's own
core rather than one optional package. The record was deliberately **not** amended in
the 2026-08-28 round — see
[[../sessions/20260828-desktop-kde-migration|the session log]] for what a re-read
found. Deciding it now means re-establishing the premise first.

**ADR-0019 is `accepted` but conditional**, and the condition is live: if KWallet
turns out to have no per-request retrieval prompt, the ratification reopens. That is
in its *Revisit when* rather than here, for the reason ADR-0006's un-noticed trigger
taught — but it belongs on this list of things that are not settled.

## When to write one

- A choice with a rationale that a future session could reasonably reverse
- A deviation from upstream guidance (ADR-0006 deviates from Anthropic's own
  devcontainer reference)
- A rejection of something already specified elsewhere (ADR-0005 rejects PRD §7.3)
- Anything where the *cost* of the choice will be noticed later by someone who
  did not see the reasoning

Diagnostics and gotchas are **not** decisions — those go in [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]].
The test: does the error message point somewhere other than the cause? Then it
is a note.
