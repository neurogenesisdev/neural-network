---
type: backlog-item
status: active
priority: low
device: arch-thinkpad
projects: [devenv]
tags: [devbox, containers, copilot, cleanup]
created: 2026-08-21T00:00:00Z
updated: 2026-08-21T00:00:00Z
commit: pending
---

# Devbox cleanup at next rebuild

> [!NOTE] Summary
> Four items to fold into the next devbox image rebuild. The first three are not worth rebuilding for on their own; item 4 arguably is, since it unblocks a documentation channel an agent uses on every task.

## Items

1. **Disable Copilot Chat.** It installed a Copilot CLI agent into the container unasked — an agent with shell access that is not in the Containerfile, will not survive a rebuild, and is not in git.
2. **Strike the vestigial `COPY`** of `~/.codex-default`. Became unnecessary once the `CODEX_HOME` relocation was adopted.
3. **Confirm the extension host is contained.** One glance at the VS Code Extensions pane — language extensions should appear under "Container", not "Local". This is the check that verifies the entire justification for using Dev Containers.

4. **`COPY` in `wiki-search` and the ArchWiki text dump.** Full reasoning and the exact paths — which are hardcoded in the script — are in [[devbox-wiki-mount]]. Listed here because it needs a rebuild, so it belongs in the same pass as the rest.

## Note

Record both agent versions in the commit message on rebuild — see [[20260818-base-image-pinned-by-digest|ADR-0007]].
