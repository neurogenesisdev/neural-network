---
type: index
created: 2026-10-21
tags: [index, decision]
---

# Decision ID allocator

> [!NOTE] Summary
> The single allocator for decision IDs across the whole vault. **Luis maintains this file.** Agents draft new records with `id: ADR-TBD` and never pick a number; Luis assigns the next free number here when he accepts a record.

## Next free number

| Next free | Reserved | Why |
|---|---|---|
| **ADR-0042** | — | — |

How to allocate: take the next free number, add the record's row below, then raise *Next free* by one.

## Erratum

ADR-0038 was assigned twice. The neurogenesis.dev i18n routing decision (2026-09-21) keeps ADR-0038. The netcup rebuild (accepted 2026-10-19) is ADR-0039 with `legacy_id: ADR-0038`, as amended on 2026-10-21. An older mention of a bare "ADR-0038" may mean either record, so read its context.

## ADR series (vault-wide)

| ID | Status | Record |
|---|---|---|
| ADR-0001 | accepted | [[20260817-rootless-podman-over-docker]] |
| ADR-0002 | proposed | [[20260816-hyprland-ipc-never-granted]] |
| ADR-0003 | accepted | [[20260818-devcontainer-agent-exposure]] |
| ADR-0004 | superseded | [[20260819-noctalia-userland-consolidation]] |
| ADR-0005 | accepted | [[20260813-dns-ownership-by-mode]] |
| ADR-0006 | accepted | [[20260816-container-storage-and-bind-mounts]] |
| ADR-0007 | superseded | [[20260818-base-image-pinned-by-digest]] |
| ADR-0008 | accepted | [[20260818-restic-exclusions]] |
| ADR-0009 | accepted | [[20260815-t5-rescue-partition-unencrypted]] |
| ADR-0010 | accepted | [[20260819-zsh-without-plugin-framework]] |
| ADR-0011 | superseded | [[20260820-thunar-retained-during-yazi-transition]] |
| ADR-0012 | accepted | [[20260818-backup-cadence-open]] |
| ADR-0013 | accepted | [[20260823-tpm2-pcr-selection]] |
| ADR-0014 | accepted | [[20260823-continuous-trim-both-layers]] |
| ADR-0015 | accepted | [[20260825-devbox-base-is-arch]] |
| ADR-0016 | accepted | [[20260825-devbox-pinning-four-layers]] |
| ADR-0017 | accepted | [[20260825-harness-state-and-identity]] |
| ADR-0018 | accepted | [[20260828-plasma-replaces-hyprland-userland]] |
| ADR-0019 | accepted | [[20260828-secret-service-provider-is-ksecretd]] |
| ADR-0020 | accepted | [[20260828-power-management-stays-with-tlp]] |
| ADR-0021 | accepted | [[20260830-no-luks-on-always-online-vps]] |
| ADR-0022 | accepted | [[20260830-bios-boot-retained]] |
| ADR-0023 | accepted | [[20260830-debootstrap-over-image-installer]] |
| ADR-0024 | accepted | [[20260830-debian-over-rhel-family]] |
| ADR-0025 | accepted | [[20260830-boot-filesystem-is-ext4]] |
| ADR-0026 | accepted | [[20260830-subvolume-layout]] |
| ADR-0027 | accepted | [[20260830-vps-holds-no-credentials-upward]] |
| ADR-0028 | accepted | [[20260830-remote-repository-is-b2]] |
| ADR-0029 | accepted | [[20260831-sshd-binds-wildcard-nftables-is-the-control]] |
| ADR-0030 | accepted | [[20260902-dotfiles-bare-git-repo]] |
| ADR-0031 | accepted | [[20260902-what-listens-and-why]] |
| ADR-0032 | accepted | [[20260902-plasma-login-path-accepted]] |
| ADR-0033 | accepted | [[20260903-agent-containment-two-lanes]] |
| ADR-0034 | accepted | [[20260911-persistent-credentials-on-netcup-vps]] |
| ADR-0035 | accepted | [[20260912-netcup-vps-is-the-coding-host]] |
| ADR-0036 | accepted | [[20260913-contained-secretary-hermes]] |
| ADR-0037 | superseded | [[20260920-public-mcp-endpoint-claude-ai]] → ADR-0041 |
| ADR-0038 | accepted | [[20260921-neurogenesis-dev-i18n-routing]] |
| ADR-0039 | accepted | [[20261018-single-operator-rebuild]] |
| ADR-0040 | accepted | [[adr-0040-vault-merge-and-agent-council]] |
| ADR-0041 | accepted | [[20261029-claude-ai-vault-connector-read-write]] |

## FDR series (historical, freelance)

FDR-001 to FDR-007 keep their IDs and files. New freelance decisions use the ADR series with `domain: freelance`.

| ID | Status | Record |
|---|---|---|
| FDR-001 | accepted | [[fdr-001-no-hosting-on-personal-infra]] |
| FDR-002 | accepted | [[fdr-002-client-pays-third-party-costs-directly]] |
| FDR-003 | superseded | [[fdr-003-separate-vault-from-infra-docs]] → ADR-0040 |
| FDR-004 | accepted | [[fdr-004-business-identity-on-neurogenesis-dev]] |
| FDR-005 | accepted | [[fdr-005-no-public-prices-until-first-paid-quote]] |
| FDR-006 | accepted | [[fdr-006-gewerbe-einzelunternehmen-kleinunternehmer]] |
| FDR-007 | accepted | [[fdr-007-bilingual-site-with-language-switch]] |
