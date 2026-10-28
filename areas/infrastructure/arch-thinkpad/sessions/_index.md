---
type: index
device: arch-thinkpad
updated: 2026-09-08
tags: [index]
---

# Sessions

Working narrative — what was tried, what failed, the order things were
discovered in, and which conclusions were revised. Everything here is the
residue that no other artifact holds.

Filenames are `YYYYMMDD-descriptive-title.md`, matching the
[[../decisions/_index|decisions/]] convention.

| Session | Subject |
|---------|---------|
| [[20260823-build-spec-schema\|2026-08-23 (1)]] | Build-spec schema settled; TPM2 unlock traced to the token plugin across three negative locations |
| [[20260823-spec-boot-promotion\|2026-08-23 (2)]] | Two churn sources needing opposite fixes; `spec/boot` promoted; lookup rule reordered after the ArchWiki turned out unreachable and the tools turned out self-documenting |
| [[20260823-spec-disk\|2026-08-23 (3)]] | `spec/disk` written from scratch; grounding its postconditions against `state/` surfaced ADR-0014 and a self-refuting PRD paragraph; a hole in the draft produced the parameter convention |
| [[20260824-spec-base\|2026-08-24 (1)]] | `spec/base` written; the execution-context boundary got a postcondition that separates four namespaces; `spec/boot` turned out to be missing `bootloader-install`, which the dangling-id count could not see |
| [[20260824-spec-containers\|2026-08-24 (2)]] | Two deferred `grep -c` aborts turned out to be ten across three chapters, two of them diagnostics no input could reach; seventh schema convention; `spec/containers` written from ADRs rather than the PRD, and `_worked-examples.md` deleted |
| [[20260828-desktop-kde-migration\|2026-08-28]] | A hand-written session target outlived Hyprland and resolved into the Plasma tree, putting a confident wrong answer into generated state; two changes arrived with a metapackage and neither announced itself; the vault turned out unable to answer "is this package installed" from any file |
| [[20260902-root-session-backlog-sweep\|2026-09-02 → 09-08]] | First root session (`sudo -i` from the desktop). A boot race the backlog had predicted fired and needed two fixes the prediction lacked; a backup unit that had never run turned out unable to; the guard passed on a network with nothing behind it; Firefox withdrawn from AppArmor by reading the profile; the lane question answered by the work and written as ADR-0033 |
| [[20260830-remote-restic-b2\|2026-08-30]] | A blocked item's premise had expired unnoticed for nine days; a 12 GiB virtual disk was found in the backup set by a human reading a progress line; the exclusion list was rewritten twice in opposite directions and the second rewrite committed the error the first had just defined |

## What belongs here

- **Dead ends.** The path not taken is invisible in the artifact that resulted,
  and expensive to re-walk.
- **Discovery order.** When a conclusion was reached, retracted, and re-reached
  for a different reason, the sequence is the finding.
- **Revised conclusions.** Especially where the original was confident.
- **Negative findings.** "It is not configured anywhere" costs more to establish
  than a positive answer and is just as easy to lose.
- **Accepted limitations with a named consequence.** Not work to be done, so not
  `backlog/`; not a choice with alternatives, so not `decisions/`. The case that
  established the category: every postcondition in `spec/` is written in a
  container that cannot execute any of it. Left unstated, that reads as an
  oversight to whoever finds it; stated, it is a constraint the method already
  accounts for.

## What does not

Everything with a home already — link it instead:

| Kind | Home |
|------|------|
| A choice a future session could reverse | [[../decisions/_index\|decisions/]] |
| An error message pointing away from its cause | [[areas/infrastructure/arch-thinkpad/state-notes\|state/notes.md]] |
| Work not done yet | [[../backlog/_index\|backlog/]] |
| A procedure with a trigger | [[../runbooks/_index\|runbooks/]] |
| What the machine currently is | [[../state/_index\|state/]] — generated |

A session log that restates a decision has moved the decision somewhere it will
not be looked for. Link, do not copy.
