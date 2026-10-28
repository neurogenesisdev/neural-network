---
type: runbook
status: active
device: shared
tags: [runbook, keepassxc, credentials, conventions]
created: 2026-10-06
updated: 2026-10-06
provenance: measured
---

# KeePassXC database: conventions and state

> [!NOTE] Summary
> The database `~/Syncthing/memory-palace.kdbx` was cleaned up and restructured on
> 2026-10-04 to 2026-10-06 (titles, notes split into entries and attributes, dead
> entries removed, new folders). This note holds the conventions for new entries
> and the tooling used. No secrets and no entry lists belong in this vault.

## Files

| File | What |
|---|---|
| `~/Syncthing/memory-palace.kdbx` | Live database, synced to the phone by Syncthing |
| `~/Syncthing/memory-palace.archive-2026-10-04.kdbx` | State before the cleanup. Needed for old logins while sending account deletion requests; delete afterwards |
| `~/dev/kp-audit` | Tooling repo: `kp_audit.py` (local LLM pass over an XML export), `kp_apply.py` (plan/apply via TSV tables) |
| `~/.local/share/kp-audit` | Working files of the cleanup. `proposals.md` and `passA.jsonl` quote note contents: delete when the review is finished |

## Title convention

| Kind | Pattern | Example |
|---|---|---|
| Normal account | `Service` or `Service - account` | `Figma`, `PayPal - luiswgordon` |
| SSH key | `SSH - <key name as in the .pub comment / on GitHub>` | `SSH - netcup-dev-infra-docs` |
| API key | `API - provider - <key name as in the provider console>` | `API - Anthropic - netcup-vps-n8n` |
| Token / bot | `Token - service - <name as issued>` | `Token - GitHub - VAULT_REPO_TOKEN` |
| Tool on a machine | `tool - machine/target` | `restic - arch-thinkpad local`, `LUKS - arch-thinkpad header backup` |

Machine names and key names are written exactly as they exist (`netcup-vps`, `arch-thinkpad`, `pixel-graphene`).

## Where a secret goes

- One entry per key set (one app, project or token as the provider issues it).
- Public ID (client ID, key ID) in Username, main secret in Password.
- Further secrets of the same set as protected attributes on that entry (Advanced, Additional attributes).
- Tokens that are revoked separately get separate entries.
- Recovery codes, PINs and security answers are attributes on the account they belong to.
- Notes hold plain information only: what uses the key, expiry, public key, fingerprint.

## Folders

`Accounts - Core`, `Business`, `Infrastructure/{arch-thinkpad, netcup-vps, pixel-graphene, Retired}`,
`Keys & Tokens`, `Dev & AI`, `Software & Cloud`, `Finance`, `Government & ID`, `Health & Fitness`,
`Home & Telecom`, `Work`, `Education`, `Shopping`, `Social`, `Games`, `Media`, `Adult`, `Travel`.

One home per entry, by what it is for. All `API -` and `Token -` entries live in `Keys & Tokens`;
SSH keys and machine secrets live under the machine in `Infrastructure`.
`SecretService` and `KeePassXC-Browser Passwords` are used by desktop apps and the browser plugin: do not move or rename them.

## Rules for scripted changes

- Never write to the live file. Scripts write a new `.kdbx`; open it in KeePassXC, check, then swap with `mv`/`cp`.
- Confirm the swap happened (file date) before treating the result as live. On 2026-10-04 the swap was skipped unnoticed and two days of edits went into the side copy.
- Plaintext exports only on tmpfs (`/dev/shm`), deleted the same day.
- KeePassXC's window could not open a file under `~/.local/share`; put files to inspect in `~/Syncthing`.
- A file written by `kp_apply.py apply` was once rejected by KeePassXC with "HMAC mismatch" while pykeepass could read it; re-saving it with `kp.save(path)` fixed it. Cause not established (commit e5bb3d4 in `kp-audit`).
- No master password on a command line or piped through `echo`.

## Open (2026-10-06)

- Remaining entries tagged `dup-check`; then strip the tags `kp-audit`, `kp-audit-new`, `dup-check`.
- New entries whose secret still sits in their notes (multi-line key sets).
- Account deletion requests (Art. 17 GDPR) for services without a delete button; list in `deleted-entries.md`, kept outside the vault.
- 115 guessed URLs from the LLM pass were never applied.
