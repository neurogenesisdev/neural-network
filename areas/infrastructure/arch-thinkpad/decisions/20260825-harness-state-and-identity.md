---
title: Harness state, config and identity in the devbox
type: decision
id: ADR-0017
aliases: [ADR-0017]
status: accepted
provenance: measured
device: arch-thinkpad
stale: false
last_verified: 2026-08-25
commit: pending
impact: high
created: 2026-08-25
updated: 2026-08-26
tags: [decision, containers, devbox, agents, state, config, credentials, identity]
constrains: spec/devenv
---

# ADR-0017 — Harness state, config and identity in the devbox

- **Status:** accepted
- **Date:** 2026-08-25
- **Provenance:** measured — the layout, the orphaned project trees and the config
  drift were all read from inside the running container 2026-08-25

## Context

The container was built for one harness. It now hosts more than one, and OpenCode and
Cline are expected. Every one of them keeps configuration, derived state, credentials
and cache — and this system has never said which is which, where each belongs, or what
happens to it on a rebuild. The result is four measured defects, none of which
announced itself.

**Project memory is keyed to the launch path.** `$STATE/claude/projects/` holds two
trees, `-work` and `-tmp`, 9.7 MB between them — one per directory the harness has ever
been launched from. Neither is reachable from the other; the earlier one is simply
orphaned. The replacement changes the mount point, which would orphan the current one
too, a third time.

**Config and state are not separated on disk.** Measured in `$STATE/claude`:

| Entry | Category | Mode |
|---|---|---|
| `settings.json`, 328 B | config | 0644 |
| `.claude.json`, 44.7 KB | config **and** state, mixed | 0600 |
| `.credentials.json`, 508 B | credential | 0600 |
| `history.jsonl`, `sessions/`, `session-env/`, `file-history/`, `plans/`, `projects/` | derived state | mixed |
| `plugins/` 21 MB, `paste-cache/`, `shell-snapshots/`, `backups/`, `.last-cleanup` | cache | 0755 |

**The seed is asymmetric and has already drifted.** The wrapper seeds Codex's config
from the repo `if [[ ! -f ]]` — once, never re-synced — and gives the other harness no
equivalent. Measured today: the repo/image copy at `~/.codex-default/config.toml` is
**214 bytes**; the live copy at `$CODEX_HOME/config.toml` is **609 bytes**, carrying a
comment block explaining the two-sandbox design that exists in exactly one place and is
not the one under version control. The drift is not hypothetical and it is invisible
from both ends.

**The permission allowlist is a file the agent can edit, and there are three copies of
it, not two.** Measured 2026-08-25/26:

| Copy | Leaves a trace? |
|---|---|
| `.claude/settings.local.json` in this repo | **yes** — tracked in git, has commit history |
| `$STATE/claude/settings.json` (user level) | no — bind-mounted state, no repository |
| `.claude.json`'s per-project `allowedTools` | no — **and excluded from restic** |

The third was found by the `jq keys` gate on 2026-08-26 and is the one this record
originally missed. It grants the same tools as the first, from a file that is not in
any repository, is not in the backup set, and is never diffed. All three are writable
by the process they constrain.

That is why `allowedTools` joins the MCP list and the trust flags in the bounded
extraction below: it is tool-granting surface with no other diff surface at all. The
same gate found that **MCP configuration is per-project rather than top-level** — there
is no `.mcpServers` key — which the first draft of the collector had wrong and which
its `# source:` marker was designed to catch.

## Decision

### 1. A stable project identity now; path parity as a separate, conditional decision

**Set `CLAUDE_CODE_PROJECT_DIR_NAME` to a stable literal.** This closes the measured
defect on its own: project memory stops being keyed to the launch directory, so no
future mount change can orphan it again.

**Path parity — mounting the project at its host-identical absolute path — is not
adopted this round.** It is deferred with a stated condition, not rejected: it is taken
when **either** the container's shell configuration exists, so the prompt marker that
mitigates the confusion it deepens is real, **or** a second harness demonstrates a
path-keyed failure the literal does not already fix.

**The ordering is load-bearing rather than merely cautious.** Taking the literal first
makes the memory key path-independent, so adopting parity later moves no project memory
at all. Reversed, each change costs its own migration and the second lands after the
first has been forgotten. **One migration, now.**

Container **user name and home** are parity of the same kind and go with it.

### 2. Four categories, and the mechanisms that can actually act on them

| Category | Destination | What enforces it |
|---|---|---|
| Config | canonical copy in the recipe repo, synced by §3 | the sync-and-diff report, and a git diff when the canonical copy moves |
| Config the tool writes and owns | a bounded section in `state/`, extracted by the collector | `git diff`, `redact.map`, the pre-commit hook |
| Derived state | restic **include** | the restic exclusion list — **narrows ADR-0008** |
| Credentials | restic **exclude** | the restic exclusion list, plus 0600 — **restates ADR-0008** |
| Cache | restic exclude | the restic exclusion list |

**`git` and `redact.map` have no jurisdiction over `$STATE/claude`.** It is not a
repository, and the pre-commit hook cannot see it. An earlier draft of this table put
"gitignored, in `redact.map`" against the credentials row and enforced the whole split
with a deny-by-default `.gitignore` — **both were mechanisms that cannot act on that
tree**, and the `.gitignore` was solving a problem that does not exist. What acts there
is the restic exclusion list, file permissions, and snapper, which captures everything
regardless and which [[20260816-container-storage-and-bind-mounts|ADR-0006]] and
[[20260818-restic-exclusions|ADR-0008]] both judge not worth another nested subvolume
to prevent.

Config reaches version control by **extraction, not by mirroring**: the collector
writes a bounded section into `state/`, which *is* in the repository and *is* covered
by the hook and by `redact.map`. Same shape as
[[../spec/base#collect-tool-versions|the tool-versions section]].

**Config mounts are not read-only.** Harnesses legitimately write into their config
directory — onboarding flags, first-run acknowledgements, theme changes made from
inside. A read-only mount converts that into an error at a moment nobody is watching
for one.

### 3. Seeding becomes a symmetric sync-and-diff

One manifest, one mechanism, every harness:

- **absent** → install from the repo.
- **present and identical** → nothing.
- **present and different** → **report the diff and change neither side.**

The third case is the whole point. Overwriting silently would have destroyed the
609-byte comment block; skipping silently is what let it drift unnoticed. Reporting
makes the human choose, which is the only correct answer when both copies are plausibly
right.

### 4. Identity markers

`--hostname devbox…`, `DEVBOX=1`, and a shell prompt that renders the hostname. Each
carries a postcondition —
[[../spec/devenv#devbox-identity-markers|`devbox-identity-markers`]].

## Rationale

**The literal closes the defect; parity generalises it — and only the first is free.**
`CLAUDE_CODE_PROJECT_DIR_NAME` is documented as a per-harness lever (*reported
2026-08-25; the lookup is one launch with it set, watching what appears under
`projects/`*). It fixes this harness, and it fixes the measured failure completely: the
two orphaned trees exist because the key is a path, and after the literal the key is
not a path. Nothing is left over for parity to repair *here*.

Parity's remaining case is **generalisation** — every harness that keys on a path,
including the ones not chosen yet, without needing to know their environment variable
names. That is a real argument and a speculative one, and it is paid for now:

- **the migration is a silent cutover.** A wrong key produces an empty memory tree,
  which reads as a fresh project rather than a lost one — no error, no diff, and the
  9.7 MB is still on disk under a name nothing looks at;
- **every spec step naming `/work` has to move** with it;
- **the wrapper's cwd guard** — the line Block-C calls "the single most valuable line
  in the script" — is written against a fixed mount point, and this vault has never
  read it;
- **and the mitigation for the confusion it deepens does not exist.** This record's own
  consequences say `archlinux:base` ships no shell configuration, so there is no prompt
  for the marker to appear in.

Shipping a change before its own mitigation is the shape this vault keeps catching in
checks: the benefit is visible, the cost is deferred and silent. So parity waits for
its mitigation, or for evidence that the literal was not enough.

**One consequence of deferring, stated so it is not a surprise:**
[[../spec/_schema#Conventions|the which-side rule]]'s preferred repair — *make the path
true on both sides* — is unavailable for project paths this round. The operative repair
is the second one, *name the side*, until parity is taken.

**The split cannot be done by environment variable, and this had to be measured to
know.** `CLAUDE_CONFIG_DIR` relocates the *whole* tree — config, state, credentials and
cache land in one directory together, which is exactly what `$STATE/claude` shows. So
the separation has to be imposed from outside the harness, and the table above names
what can impose it.

**`.claude.json` cannot be split as a file, which is not the same as being
undiffable.** It mixes MCP server definitions and per-project trust (config) with the
OAuth account and counters (state) in one 44 KB file, and ADR-0006 records why it
cannot simply be left outside the mount — `CLAUDE_CONFIG_DIR` is what moves it inside,
and without that you are signed out every exit. The file therefore goes to the state
destination. **But the vault owns a third option the draft missed: the collector.** A
bounded section in `state/` carrying MCP server names and transports and the
per-project trust list — and nothing else — puts changes into the git diff without the
44 KB file, or the account inside it, entering the repository. MCP servers are
tool-granting surface, so a change to which ones this project trusts is not cosmetic
and should not be invisible. [[../spec/base#collect-harness-config|`collect-harness-config`]]
is the step.

**Extraction is an allowlist of fields, not a redaction of a dump.** A redaction has to
know every field that might carry an identifier, in a file this system does not control
and whose schema can gain one at any release — it fails open. An allowlist gains
nothing when the file gains a field, which is the correct direction to fail.

**A file inside the container cannot constrain the container.** The permission
allowlist is advisory in the strict sense: the process it governs can rewrite it, and
the user-level copy leaves no trace when it does. This is the same class of problem the
egress work addresses and the opposite kind of answer — an nftables rule written by a
`NET_ADMIN` sibling into a netns the agent has no capability to touch is a control the
agent cannot edit. **The allowlist is a workflow aid; the egress rules are a boundary.**
Recording that distinction is the point, because they look alike from inside and are
nothing alike.

## Consequences

- **This record removes no tell at all.** With path, user and home parity deferred,
  `whoami` still returns `dev`, `$HOME` is still `/home/dev`, and `pwd` in the project
  is still `/work`. Three of the four tells survive; the one that goes is the distro,
  and [[20260825-devbox-base-is-arch|ADR-0015]] removes that one.
- **The full loss belongs to the deferred decision, and is one of the reasons it is
  deferred.** When parity is taken, nothing about a prompt, a `pwd`, a `whoami` or an
  `/etc/os-release` distinguishes the sides — and the mitigation for that is the shell
  prompt, which does not exist. The markers are what this record contributes to a cost
  it does not yet incur: `/run/.containerenv` for checks (*measured present*), and
  `--hostname devbox…` plus `DEVBOX=1` for humans, ready before they are needed.
- **The prompt does not exist yet.** `archlinux:base` ships no shell configuration, so
  there is no prompt for a marker to appear in. Until that is written the human-facing
  mitigation is absent and the machine-readable one is carrying it alone. Open item,
  named rather than assumed.
- **Enterprise policy at `/etc/claude-code/CLAUDE.md` is deliberately not persisted.**
  Measured: the path does not exist. It is container-only by design, so its absence is
  a decision and not an oversight — stated here so nobody adds a mount for it later on
  the assumption that something was forgotten.
- **`settings.json` is 0644 with content, and ADR-0006's own trigger has fired.** That
  record says "22 bytes (defaults) and mode 0644 while its neighbours are 0600.
  Harmless while empty; tighten if it gains content." It is 328 bytes and still 0644.
  The content is a model name, a theme and a plugin list — not secret — but the
  condition the ADR set has been met and nothing happened, which is the part worth
  recording.
- **The orphaned `-work` tree needs migrating, and `-tmp` needs a decision.** Doing
  nothing orphans 9.7 MB of project memory a third time. Migration is a one-time move
  of `projects/-work` onto the new key; `-tmp` is a launch from `/tmp` and is almost
  certainly disposable, but deleting it is a choice someone should make rather than a
  side effect. **The migration target depends on how the stable literal composes the
  key** — whether it replaces the whole directory name or one component of it — which
  is the lookup named above and must be answered before the move, not after.
- **The sync-and-diff will report on first run**, because the drift already exists.
  That is the mechanism working, not a fault, and the 609-byte version is the one to
  keep.
- **A rebuild no longer silently loses local config edits** — it reports them. That is
  the direct repair of the failure this record was written for.

### This closes ADR-0008's open question rather than inventing a policy

**[[20260818-restic-exclusions|ADR-0008]] already asked exactly this, on 2026-08-21,
and left it open:**

> Claude Code session history is not agent state in the disposable sense —
> `~/.local/state/devbox/claude/` holds `sessions/`, `projects/`, `history.jsonl` and
> `plugins/` alongside `.credentials.json`, and the current exclusion discards all of
> it. If that history is worth keeping, **narrow the exclusion to the credential files
> rather than the whole directory.**

It is on `_index.md`'s open-questions table and has a backlog item. So this record does
not get to write a restic policy from scratch; it answers a question already asked, and
amends ADR-0008 rather than quietly disagreeing with it.

**Narrow the exclusion. Derived state in, credentials and cache out.**

    # was:   --exclude '/home/luiswg/.local/state/devbox'
    --exclude '/home/luiswg/.local/state/devbox/claude/.credentials.json'
    --exclude '/home/luiswg/.local/state/devbox/claude/.claude.json'
    --exclude '/home/luiswg/.local/state/devbox/claude/plugins'
    --exclude '/home/luiswg/.local/state/devbox/claude/paste-cache'
    --exclude '/home/luiswg/.local/state/devbox/claude/shell-snapshots'
    --exclude '/home/luiswg/.local/state/devbox/claude/backups'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/auth.json'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/log'
    --exclude '/home/luiswg/.local/state/devbox/codex-state/tmp'

**Credentials stay excluded, and the argument that decides it is not the one about
exposure.** For inclusion: the tokens are already on LUKS, restic encrypts client-side,
and the marginal exposure is genuinely small. Against: re-authenticating costs about
thirty seconds. On those two alone it is nearly a coin toss, and ADR-0008 settled it
that way — "backing them up puts long-lived credentials in the repo for no recovery
benefit."

The decisive argument is one ADR-0008 does not make, and it is about **revocation**. A
restic repository's retention is not a credential's lifetime. A token deliberately
revoked after a suspected compromise stays in every snapshot taken before the
revocation, and a semi-automatic restore re-materialises it. That converts "revoked"
into "revoked unless someone restores," which is not a property anyone chose. Thirty
seconds of re-authentication is a small price for not having that.

**`.claude.json` is excluded with the credentials**, because it carries the OAuth
account. That is affordable *only because* `collect-harness-config` extracts the MCP
server list and per-project trust into `state/` — the record survives even though the
file does not. Without the collector section this exclusion would silently discard
configuration; with it, the two halves interlock.

**Residual, named and accepted: restic is exclude-based, so this list fails open.** A
harness that invents a new credential file is backed up until someone notices. A
deny-by-default equivalent would mean curating an include list of everything worth
backing up on the whole machine, which is a much larger change than this record should
make. The shape that would close it is an enumeration check like
`devbox-ephemeral-writes`, if it ever becomes worth writing.

### The transcripts are the larger residual, and the same argument reaches them

Narrowing brings `sessions/`, `projects/` and `history.jsonl` into the repository.
Those are a **verbatim record of everything typed and pasted into the container across
every session** — including command output pasted back during probe rounds, which is
precisely how a token, a key or an unredacted identifier gets into one. `redact.map`
and the pre-commit hook have no jurisdiction over them, as this record establishes
above.

**So the argument that decided the credentials row applies here too**, one layer over
and across a much wider surface: a restic repository's retention is not a secret's
lifetime, and a revoked credential that appears in a transcript is re-materialised by
the same semi-automatic restore. Stating the risk for `.credentials.json` and saying
nothing about the thing most likely to contain a copy of it would be the more careful
half of a decision.

**It is accepted rather than narrowed, and the reason is that narrowing it here would
undo the decision.** The history *is* what ADR-0008 asked to keep; excluding `projects/`
returns to the directory-wide exclusion this narrowing exists to replace. Two things
make acceptance defensible rather than merely convenient:

- **The original is still excluded.** What can reach a transcript is a *copy* — pasted
  or echoed — not the credential store itself. The exposure is real but is not the
  systematic one the credentials row closes.
- **restic encrypts client-side**, so the surface is a restore or a repository
  compromise, not the transfer or the T5 sitting in a drawer.

**The mitigation is behavioural, and writing it down is the point.** *Do not paste
credentials into a session*, and *treat a restore of `projects/` as containing
secrets.* Both are current practice and neither has ever been recorded as a
backup-scoped requirement, which is how a practice becomes an assumption and then stops
being followed.

**No detector is proposed, deliberately.** Scanning transcripts for secret-shaped
strings has a false-negative problem the vault has already ruled on: a scan that
reports clean is worse than no scan, because it converts an accepted risk into a
believed-absent one. `_schema.md`'s rule about checks that fail toward all-clear covers
it.

- **MCP configuration and per-project trust remain undiffable *as a file*** — the 44 KB
  `.claude.json` is not going into the repository — **but they are now diffable as a
  record**, through `collect-harness-config`. What is genuinely lost is everything else
  in that file: onboarding flags, counters, tips-shown state. None of it is worth a
  diff.

## Revisit when

A harness arrives that keys project memory on something other than a path, or one that
cannot be configured from a file at all. Either breaks the manifest model, and the
manifest is the part designed to absorb new harnesses without new decisions.
