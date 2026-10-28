---
type: backlog-item
status: active
priority: medium
device: arch-thinkpad
projects: [devenv, vault]
tags: [devbox, containers, documentation, lookup]
created: 2026-08-23T00:00:00Z
updated: 2026-08-23T00:00:00Z
commit: pending
---

# Mount the local ArchWiki read-only into the devbox

> [!NOTE] Summary
> `arch-wiki-lite` and `arch-wiki-docs` are installed on the host but not visible inside the devbox container, so an agent working in the vault cannot run `wiki-search` and has to round-trip every wiki lookup through the human.

## The gap

The root `CLAUDE.md` lookup rule names the local ArchWiki as channel 3, after
the tool's own help and `man.archlinux.org`. Channel 3 is currently unreachable
from where the work happens: `/usr/share/doc/arch-wiki*` does not exist in the
container (measured 2026-08-23), and `wiki-search` is not on `PATH`.

This matters more than it looks, because channel 3 is the *fallback*. The live
ArchWiki is behind an Anubis proof-of-work interstitial and cannot be fetched by
any non-browser client (see [[areas/infrastructure/arch-thinkpad/state-notes|notes.md]]). So when the man page
does not cover something — `mkinitcpio.conf(5)` documenting the `HOOKS` array
but not a single hook is the case that prompted this — there is no fallback
inside the container at all. The 2026-08-23 session lost a turn to exactly that
and closed the gap only because the human pasted `mkinitcpio -H` output back in.

## What to do

**Two parts, not one — this is not wrapper-only.** `wiki-search` is a bash
script at `/usr/bin/wiki-search` on the host, owned by `arch-wiki-lite`. It is
not in the image, so mounting the data alone gets nothing runnable. Either
`COPY` the script in the Containerfile, or bind it read-only next to the data:

```
--volume /usr/share/doc/arch-wiki/text:/usr/share/doc/arch-wiki/text:ro
--volume /usr/bin/wiki-search:/usr/bin/wiki-search:ro
--env    wiki_width=100
```

**The path is hardcoded and cannot be relocated.** The script sets
`wiki_path="/usr/share/doc/arch-wiki/text"` and
`html_path="/usr/share/doc/arch-wiki/html"` directly. The mount has to land at
exactly that path; there is no environment variable or flag to point it
elsewhere.

**Mount `text/` only.** `pacman -Ql` shows a clean split between the two
packages: `arch-wiki-lite` owns `text/arch-wiki.txt.gz` and `text/index` at
about 20 MB, `arch-wiki-docs` owns `html/` at about 225 MB. Binding just `text/`
gets `wiki-search` working at a tenth the size. The consequence is explicit:
**`wiki-search-html` will not work**, because `html_path` will not exist. That
is deliberate — the text dump is what an agent greps, and the HTML tree is worth
neither the bytes nor the container start time.

**`wiki_width=100` is not cosmetic.** The script calls `stty size` to get the
terminal width and falls back to 80 when it returns nothing. In a non-tty
context — which is every agent invocation — `stty` emits a warning on stderr
before the fallback takes effect. Setting `wiki_width` in the environment skips
the `stty` call entirely, so the output is clean rather than clean-after-a-
spurious-error. An agent that has been taught to treat stderr as signal will
otherwise read that warning as a failed lookup.

**Read-only is the point, not a detail.** ADR-0006 binds project and agent state
from `@home` because those need to persist and be snapshotted. This is the
opposite case: reference data the container should read and can never
meaningfully write. `:ro` also means it cannot become another unversioned
writable path.

## Check before adopting

**Version lag is inherited, not fixed.** Both packages are at `20260702`,
roughly seven weeks behind live. Mounting them does not make them current, and
the lookup rule's ordering — tool's own help first, man page second — still
stands for exactly that reason.

## Related

- [[devbox-contents-audit]] — **the general case.** This item is one instance of
  it; the audit covers the man page equivalent, which wants the same mount
  pattern and the same rebuild.
- [[devbox-cleanup-next-rebuild]] — **the natural home for this.** Since the
  script has to come in as well as the data, `COPY`ing it in the Containerfile
  is the cleaner of the two routes, and that means a rebuild either way. Fold
  this in rather than treating it as separate work.
- [[20260816-container-storage-and-bind-mounts|ADR-0006]] — the bind-mount
  reasoning this follows.
