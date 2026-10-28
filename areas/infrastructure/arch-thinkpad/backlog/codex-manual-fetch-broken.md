---
type: backlog-item
status: active
priority: low
device: arch-thinkpad
projects: [devenv]
tags: [devbox, codex, documentation, egress]
created: 2026-08-28T00:00:00Z
updated: 2026-09-03T00:00:00Z
commit: pending
---

# Codex manual fetch helper has no help interface

> [!NOTE] Summary
> The OpenAI Docs skill ships a manual-fetch helper, but its CLI rejects
> `--help`. An agent probing the helper in the usual tool-first order therefore
> read its source and fell back to web search even though the supported no-argument
> invocation works. Fix the probe path or make the failure lead to the bundled
> usage documentation rather than to network access.

> [!INFO] 2026-09-03 — what the failure is and is not
> Not the container: the helper is in Codex's own skill directory, which is
> bind-mounted, and the documented no-argument invocation works from inside. The
> helper simply has no `--help`, and an agent following the vault's tool-first
> probe order hits that wall first. The fix is on the agent side — probe with the
> documented invocation from `references/codex-self-knowledge.md` — or upstream.
> Folded into the rebuild pass per [[../decisions/20260903-agent-containment-two-lanes|ADR-0033]].

## What was observed

The failure was first reported from the Arch image on 2026-08-27
(`inherited-unverified` as to that date). It was reproduced in the current devbox
on 2026-08-28 (`measured`):

```console
$ node /home/dev/.codex-state/skills/.system/openai-docs/scripts/fetch-codex-manual.mjs --help
Error: Unknown argument: --help

Details:
ManualFetchError: Unknown argument: --help
    at parseArgs (.../fetch-codex-manual.mjs:518:13)
    at main (.../fetch-codex-manual.mjs:533:16)
    at .../fetch-codex-manual.mjs:585:3
```

The remaining stack includes Node's ESM loader. The rejection is caught by the
helper's own `main().catch(...)`; describing it as an unhandled rejection would
be an `inferred` claim and is not supported by the output or source inspected on
2026-08-28.

The broader statement that the helper cannot be invoked is also not supported.
The skill's bundled `references/codex-self-knowledge.md` gives the supported
invocation with no arguments. That invocation, with the documented
`--status-json` option added, completed successfully in the same devbox on
2026-08-28 and reported a current manual at
`/tmp/openai-docs-cache/codex-manual.md` (`measured`):

```console
$ node /home/dev/.codex-state/skills/.system/openai-docs/scripts/fetch-codex-manual.mjs --status-json
Manual status: local manual was already current.
```

So the defect is the missing discoverability/error-recovery interface, not a
demonstrated failure of manual fetching.

## Why it matters for egress

The root `CLAUDE.md` lookup rule asks the installed tool first. Here the usual
`--help` probe could not answer. The agent recovered by reading the shipped
argument parser, then used web search. The second step was network-backed even
though the helper and a cached copy of the manual ship or persist locally.

That recovery path is coupled to the pending egress round (`inferred`): under an
allowlist, web fallback either fails or requires `developers.openai.com` to be
allowed. Neither policy has been chosen. Cross-reference
[[../decisions/20260825-devbox-base-is-arch|ADR-0015]], whose “What this record
does not touch” section explicitly leaves the container-netns egress ADR for the
next round. [[devbox-wiki-mount]] is the parallel local-channel problem for the
ArchWiki lookup path.

## Scope still unknown

Whether rejection of `--help` is specific to the Arch image is `inferred`, not
measured. The Debian image is gone, so it cannot provide a direct comparison.
The lookup that closes this gap is the same command on a fresh Codex install
outside this container, with its Codex/OpenAI Docs skill version recorded.

## Subdirectory agent instructions established in the exchange

Two scoped instruction files exist as symlinks (`measured` 2026-08-28):

- `arch-thinkpad/decisions/AGENTS.md` -> `CLAUDE.md`
- `arch-thinkpad/state/AGENTS.md` -> `CLAUDE.md`

The bundled Codex manual says nested `AGENTS.md` files apply more specifically
within their subtree. It follows that these symlinked files combine with the
root `/work/AGENTS.md`, with the deeper file governing its subtree (`inferred`:
the documented nested-file rule and normal symlink resolution establish the
conclusion, but no conflicting instruction was deliberately exercised to
demonstrate enforcement).

## Done when

- `--help` prints the helper's accepted options and exits successfully, **or**
  the OpenAI Docs skill documents a different local probe that an agent can use
  before inspecting source;
- a failed probe directs the agent to the bundled invocation documentation or
  cached manual before any web fallback;
- the fresh-install comparison records whether this belongs to the image or to
  the shipped skill; and
- the pending egress round explicitly decides whether Codex documentation web
  fallback, including `developers.openai.com`, is allowed.
