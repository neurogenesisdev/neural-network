# Infrastructure — scoped contract
Read areas/infrastructure/_index.md and the relevant device and type indexes first.
Before changing a design, read the accepted ADRs that apply and their later amendments.
For Arch, check the decision index: ADR-0002 limits AppArmor hypr/** grants, ADR-0005 rejects the old DNS design, ADR-0006 rejects podman named volumes.
Mark every claim measured (command run and seen), inferred (reasoned) or inherited-unverified (not rechecked).
Use those markers in prose and in frontmatter. Never promote an inference to a measurement.
Ground machine claims in current collector output, not in the superseded _source/ documents.
If state cannot confirm a claim, label it inferred or make a fresh, authorized observation.
The VPS has no collector. Its measurements are dated observations, not continuous verification.

## Generated state
Everything under arch-thinkpad/state/ is collector output. Never hand-edit it; fix the machine or the collector and rerun through its owner.
The exception is arch-thinkpad/state-notes.md: curated commentary on state, kept outside state/ and edited by hand.

## Look it up, never recall it
Look up command flags, config syntax and current tool behavior before writing them.
First ask the installed tool: its --help, subcommand or hook help, or introspection.
Then the packaged man page (man.archlinux.org for Arch), then the local ArchWiki.
For upstream projects not covered there, use their official docs or Context7's exact project match.
A fuzzy match to a different tool is no evidence. Do not use it.
The Anubis block on the live ArchWiki and the local wiki's age are dated observations. Do not retry blindly; recheck when it matters.
If the lookup fails, state the intended result, mark it inferred, and name the gap and the lookup that would close it.
Postconditions resolve values from live state or generated artifacts, never from copied UUIDs or assumed paths.
Use blkid for current UUIDs, and inspect the real boot artifacts after generating them.

## Redaction
Never commit redact.map or substituted scripts. Redact rather than bypass the hook.
Use the established MACHINE_ID, TS_IPV4, TS_IPV6, FSUUID and serial placeholders in prose too.
Public site addresses can slip past the pattern set; substituting them stays a controlled step.
This vault's pre-commit gate is infra-tools/.githooks, set through core.hooksPath. Never use --no-verify.
Executable hooks and generators live in infra-tools or the admin install; notes link to that source.

## Decisions
Accepted Decision and Rationale sections are append-only, and decision files are never deleted.
Reverse a decision with a new record plus superseded_by. Mark additions with a dated "Amended YYYY-MM-DD:" line.
Use _templates/tpl-decisions.md with id: ADR-TBD; Luis assigns the number. Never take an area's next local number.
Existing dated ADR filenames and FDR IDs stay unless a separately reviewed correction changes them.
ADR-0038 is the neurogenesis.dev i18n routing decision (2026-09-21). The netcup rebuild is ADR-0039, legacy_id ADR-0038, by the erratum of 2026-10-21.
Resolve any older bare "ADR-0038" mention by its context before relying on it.
For the VPS, keep ADR-0021's accepted no-LUKS trade-off. Do not transplant the laptop threat model.
Keep ADR-0027's no-upward-credentials rule. Per-repo GitHub and backup keys are not upward access.
Keep ADR-0029: sshd binds the wildcard and nftables controls reachability. Do not add a Tailscale ListenAddress dependency.
Where older decisions conflict, follow ADR-0039's later principal and credential amendments.

## Build scripts
Build scripts live in infra-tools; its CLAUDE.md holds the guards. Never move them back into the vault.
Do not write a runnable lockdown script before its human reachability prerequisite is met.
Keep the committed, corrected rebuild script distinct from the historical version that actually ran.
