---
title: "neurogenesis.dev — site operations (repo, build, deploy, legal pages)"
type: runbook
provenance: measured
device: shared
stale: false
last_verified: 2026-09-24
commit: pending
created: 2026-09-15
updated: 2026-09-25
tags: [runbook, website, astro, cloudflare-pages, github-actions, legal, neurogenesis.dev, freelance]
---

# neurogenesis.dev — site operations

> [!NOTE] Summary
> Astro static site on Cloudflare Pages, repo `neurogenesisdev/neurogenesis-dev`
> (renamed from `neurogenesis-blog`; GitHub owner renamed from `Luis-Gordon`,
> both 2026-09-15), worked on from netcup-vps as
> `dev`. Business section (FDR-004) went live 2026-09-15: /impressum,
> /datenschutz, /contact; /services is built but unlinked and `noindex` until
> the Mentor's gate opens. Deploys run in GitHub Actions on push to `main`;
> no Cloudflare token exists on the VPS. Business-side copy decisions:
> `freelance-practice/business/web-presence.md`.

## 1. Where things are

| Thing | Location |
|---|---|
| Repo | `github.com/neurogenesisdev/neurogenesis-dev` (private); old owner/name redirect until someone claims them |
| Clone (coding host) | netcup-vps, `dev`, `~/dev/neurogenesis-dev`; deploy key `~dev/.ssh/gh-neurogenesis-dev` (write), alias `gh-neurogenesis-dev` (ADR-0034 row) |
| Obsidian vault of the project | inside the repo: `neurogenesis-dev/` (renamed from `neurogenesis-vault/`); laptop copy `~/dev/neurogenesis-dev`, synced by `ob sync` — tooling only, the build does not read it |
| Hosting | Cloudflare Pages project **`neurogenesis-dev`** (Direct Upload, `neurogenesis-dev.pages.dev`, created 2026-09-15); custom domains `neurogenesis.dev` (canonical, `site:` in `astro.config.mjs`) and `www`, both CNAME → `neurogenesis-dev.pages.dev`, proxied. Old project `neurogenesis-dev-old` (`neurogenesis-blog.pages.dev`; renaming a project does not change its pages.dev address) — delete after 2026-09-22 |
| DNS / email | [[neurogenesis-dev-dns-and-email]] |
| Stack | Astro 6.4 static, `@astrojs/mdx`, Tailwind v4, Pagefind search, satori OG images, self-hosted Inter/JetBrains Mono, Cloudflare Web Analytics (cookieless), Kit newsletter form, Mermaid from jsDelivr at runtime |
| Content | `content/blog/*.mdx`, `content/projects/*.mdx`, `src/content/legal/datenschutz.md` (legal collection); pages in `src/pages/` |
| Agent rules | repo `CLAUDE.md` → "Agent notes" section (sandbox quirks, build gate, known check error) |

## 2. Build and verify (measured 2026-09-14/15)

Plain shell as `dev`: `npm ci && npm run build && echo BUILD-OK` — the build
runs three `tsx` pre-scripts (link previews, KaTeX assets, OG images), `astro
build`, then Pagefind. Build-time env: `PUBLIC_CONVERTKIT_FORM_ID` comes from the GitHub **repository variable** of the same name (all three workflows); without it the signup form renders nothing. **Never pipe the build into `tail`/`head`** — the pipe
hides a failed exit code (that is how a broken lockfile got committed once).

**Since 2026-09-24 the Claude Code sandbox is off on netcup-vps**
(infra-docs `netcup-vps/runbooks/coding-rig-stage-1.md` §Amendments), so this
sandbox caveat no longer applies; kept for history. Inside the old sandbox
`npm run build` failed (tsx's IPC socket); the gate there was `node --import tsx` for the three scripts + `astro build` +
pagefind + `npm run check` + `npm run lint`. `npm run check` has one known
pre-existing error (`src/plugins/rehype-sidenotes.ts`, `ariaHidden` typing) —
any other error is real. The sandbox also lists home dotfiles as untracked in
the project root; they are not in the repo — never `git add -A`.

Dependency rule: update within ranges (`npm update`) freely; majors one at a
time. `@tailwindcss/vite` resolves its peer to Vite 8 while Astro 6 ships
Vite 7 → build error `Missing field tsconfigPaths …`; fixed by pinning
`vite@^7` as a devDependency (2026-09-14). The build rewrites
`public/katex/katex.min.css` after a katex bump — commit it with the bump.
Astro 7.3 exists; the upgrade is its own session (migration guide, three
custom remark/rehype plugins, `markdown.remarkPlugins` deprecation warning).

## 3. Deploy

`.github/workflows/deploy.yml` (2026-09-15): push to `main` → `npm ci`,
`lint:content`, build, `wrangler pages deploy dist --project-name
neurogenesis-dev --branch main`; `workflow_dispatch` with an optional `branch`
input → same build, `--branch staging` (preview at
`staging.neurogenesis-dev.pages.dev`). **Run it with "Use workflow from" set to
the branch under test** — the dropdown picks the workflow file version.
**The `branch` input is the git branch to check out and build, not the target
— the target is always staging.** Typing `staging` there fails at checkout
(measured 2026-09-23). The task branch must be pushed first
(`git push -u origin site/<task>`) or it is not in the dropdown; delete it on the
remote after merging.
Secrets `CLOUDFLARE_API_TOKEN` (account token `github-actions-neurogenesis-dev`,
Pages Read/Write only, expires 2027-09-18, copy in KeePassXC),
`CLOUDFLARE_ACCOUNT_ID` live in the GitHub repo only. `gh` is not installed on
the VPS — dispatch from the web UI. Concurrency per target,
in-flight deploys never cancelled. First run measured green; four new pages
200 within two minutes.

**Vault Publish** and **Daily rebuild** were fixed and re-enabled 2026-09-19
(see change log). Vault Publish runs on `repository_dispatch` from the drafts
repo and on manual trigger; Daily rebuild runs on a nightly cron and releases
timed `publishDate` posts.

**Standard (Mentor, 2026-09-19): after every build/deploy change, test the
newsletter form on the live URL the same day** — `curl -s
https://neurogenesis.dev/ | grep -c data-form-id` must print ≥ 1, plus one
look in the browser.

### Build-time config and secrets (owner: Luis; reminders 2 weeks before expiry)

| Name | Where | What | Expires / reminder |
|---|---|---|---|
| `CLOUDFLARE_API_TOKEN` | repo secret | CF account token `github-actions-neurogenesis-dev`, Pages R/W only; KeePassXC copy | 2027-09-18 / 2027-09-04 |
| `CLOUDFLARE_ACCOUNT_ID` | repo secret | account id (not sensitive) | — |
| `VAULT_REPO_TOKEN` | repo secret | fine-grained PAT, contents:write on neurogenesis-dev + neurogenesis-drafts (checkout drafts, push posts); KeePassXC copy | 2027-09-17 / 2027-09-03 |
| `BLOG_REPO_TOKEN` | **drafts** repo secret | fine-grained PAT, contents:write on neurogenesis-dev (repository_dispatch); KeePassXC copy | 2027-09-17 / 2027-09-03 |
| `PUBLIC_CONVERTKIT_FORM_ID` | repo **variable** | Kit form id, public, baked in at build | — |

Manual fallback: `npx wrangler login` on the laptop, `npm run deploy:staging`
/ `npm run deploy`. Keep it manual — no token on the VPS (ADR-0034 property 5).

## 4. Legal and business pages (live 2026-09-15)

- `/impressum` — § 5 DDG, Einzelunternehmer, home address, no phone, no VAT
  line, § 18 MStV line, VSBG sentence. Optional Geschäftsbezeichnung line
  "neurogenesis.dev" under the name.
- `/datenschutz` — German, canonical; rendered from
  `src/content/legal/datenschutz.md` (frontmatter `updated`). Generated
  2026-09-15 with datenschutz-generator.de (free licence: valid while revenue
  < €5,000/yr — **premium licence or regeneration when that changes**), then
  hand-edited: whistleblower module and CMP bullet removed, processors added
  (Cloudflare hosting + Web Analytics + Email Routing, Google Gmail,
  Kit, jsDelivr, Cal.com), two-click consent stated. **Rule: any new
  third-party script or service → same commit updates this file.** English
  `/privacy` stays as a courtesy translation with a pointer.
- `/contact` — `ObfuscatedEmail.astro` (data attributes + tiny script,
  `noscript` fallback) and a Cal.com button from `BOOKING_URL` in
  `src/lib/constants.ts` (link, no embed).
- `/services` — **unlinked, `<meta name="robots" content="noindex, nofollow">`,
  excluded from the sitemap** (`sitemap.filter` in `astro.config.mjs`) until
  the Mentor's gate: employer consent in Textform + Gewerbeanmeldung /
  Fragebogen filed. Go-live = remove the meta, drop the filter, add the nav
  link. Copy and prices per `web-presence.md` (no public prices).
- Embeds: `YouTube.astro` (youtube-nocookie) and `Tweet.astro` load nothing
  until the visitor clicks the placeholder (measured: no `youtube-nocookie.com`
  iframe in built HTML, only in the click handler's data attribute).
- Footer links Impressum · Datenschutz · Kontakt.

Google OAuth consent screen (n8n Gmail classifier) now points its privacy
policy at `/datenschutz`; branding verified 2026-09-15 after the Search
Console TXT record and renaming the app to `neurogenesis.dev`. Scope
verification (`gmail.modify`) stays unverified on purpose — CASA assessment,
not worth it for a one-user app.

## 4b. Redesign round one — what changed (2026-09-22 → 24)

Plan and per-task record: `freelance-practice/business/site-redesign-round-one.md`
(T1–T7 + visual reset merged). Routing decisions: [[../decisions/20260921-neurogenesis-dev-i18n-routing|ADR-0038]].

- **Routes:** EN at `/`, DE under `/de/` (Astro built-in i18n, `LangSwitch`,
  hreflang + sitemap alternates, `scripts/check-i18n-twins.mjs` in lint).
  `public/_redirects`: `/datenschutz` → `/de/datenschutz`, `/blog` → `/articles`,
  `/projects` → `/work`, `/projects/*` → `/work` (all 301; project detail pages removed in T6 — projects are entries on /work only, frontmatter `title/summary/role {en, de}`). Posts at `/articles/<slug>` since T7 (`/blog/series/*` and `/blog/*` 301, series rule first). Search (Pagefind modal) only on /articles and posts.
  Legal pages now: `/impressum` (EN) + `/de/impressum`, `/privacy` (EN) +
  `/de/datenschutz` (binding). Google OAuth consent screen still points at
  `/datenschutz` — works via the 301; update it when convenient.
- **/services gate:** `SERVICES_LIVE=false` in `src/lib/constants.ts` drives the
  noindex meta, the sitemap filter and the hidden nav item for `/services` and
  `/de/services`. Go-live = flip that constant (replaces the manual steps in §4).
- **Design tokens / contrast:** `src/styles/global.css`; every text/background
  pair is checked by `scripts/check-contrast.mjs` in `npm run lint`. Links on
  light bands `ink-0` underline, on dark bands `accent-text`.
- **Visual workflow:** `scripts/shot.sh <path|url> <WxH> [out]` (serves `dist/`
  on :4173) or the Playwright MCP; output in `.shots/` (gitignored). Reference
  screenshots the agent must match live in `references/` (committed).
  To look at them on the laptop: `scp -r vps-dev:dev/neurogenesis-dev/.shots ~/Downloads/shots`.
- **`.claude/` is gitignored** in this repo — project settings are local to the clone.

## 5. Backlog (site)

- Redesign round one — **planned 2026-09-21**: brief `freelance-practice/business/site-design-brief.md`, task list `…/site-redesign-round-one.md` (T0–T10, cap 15 h), routing [[../decisions/20260921-neurogenesis-dev-i18n-routing|ADR-0038]]. Changes this runbook when shipped: `/datenschutz` → `/de/datenschutz` (301; update the Google OAuth consent-screen privacy URL), `/blog` → `/articles` index (post URLs unchanged), `/projects` → `/work`, `SERVICES_LIVE` flag in `constants.ts` replaces the manual gate steps in §4, Brevo added to both privacy files, OFL licence files in `public/fonts/`.
- Vault content inside `neurogenesis-dev/` still has ~65 notes referencing
  the old folder/repo names — laptop `sed` with Obsidian closed.
- Astro 7 upgrade; `@astrojs/mdx` 8, katex 0.18, web-vitals (delete
  `WebVitals.astro` instead); fix `rehype-sidenotes.ts` type error.
- Render Mermaid at build time instead of jsDelivr at runtime, then tighten
  `public/_headers` CSP and drop jsDelivr from the Datenschutzerklärung.
- Case study collection: `content/projects/` for the yoga site once live and
  consented.
- Pipeline hardening (optional): script treats missing `publish/blog/` as
  "nothing to publish, exit 0"; deleting a post in drafts should remove its
  MDX (finding K). Recreate the Obsidian→drafts `ob sync` cron when blogging
  resumes.
- Naming (2026-09-15): repo, clone dir, in-repo vault folder and laptop vault path are all `neurogenesis-dev` → path `~dev/dev/neurogenesis-dev/neurogenesis-dev/`, and `~/dev/neurogenesis-dev` means the repo on the VPS but only the vault on the laptop. Proposed: rename the in-repo folder to `vault/` in one session with Obsidian closed and `ob sync` reconfigured; brand names (domain, handle, repo, Pages project) stay.
- Cleanup done 2026-09-15: Pages project `neurogenesis-dev-old`, tokens `Edit Cloudflare Workers`, `memogenesis build token`, `tmp-domain-move` deleted. If the SaaS (memogenesis) CI deploys to Cloudflare, it needs a new scoped token. Branches `chore/deploy-workflow`, `chore/rename-neurogenesis-dev` deleted.

## 6. Change log

- 2026-09-25 — T7 merged: articles + legal pages on the light bands, `/blog/*` → `/articles/*`, Brevo listed in both privacy files. T7b (same day): Giscus removed — no comments on the site; Giscus and the generator comments module gone from both privacy files, `giscus.app` gone from the CSP; Brevo added.

- 2026-09-24 — T6 /work merged (project list EN+DE, `/projects/*` splat, BaseLayout `fillMain` prop). Form check live = 1.

- 2026-09-22 → 24 — Redesign round one T1–T5 + visual reset merged to main
  and deployed (greyscale/blue tokens + contrast lint; layout shell without
  ClientRouter; bilingual routing; home; /services EN+DE still gated; Huberman
  logo box, colour bands, centred layout, one booking button, new footer).
  Newsletter form checked live after each deploy (`grep -c data-form-id` = 1).
  Detail in §4b and the round-one note.

- 2026-09-17/19 — Publish pipeline fixed (branch fix/publish-pipeline, merged;
  commits c121b67, 3a46752, e57dd8e, 360bbc2 + AC-22b fix 1076766).
  Original failure: both fine-grained PATs had expired (finding D) — daily
  cron died at the drafts checkout. Design flaw fixed: vault-publish's own
  deploy removed; it now pushes with VAULT_REPO_TOKEN so deploy.yml is the
  sole deployer. scheduled-publish → "Daily rebuild" (cron, production
  concurrency group; releases timed publishDate posts). 4 test posts
  deleted (drafts source first — finding K: the script never deletes).
  New PATs: VAULT_REPO_TOKEN contents:write on neurogenesis-dev +
  neurogenesis-drafts; BLOG_REPO_TOKEN contents:write on neurogenesis-dev
  (secret in drafts). Both expire 2027-09-17; copies in KeePassXC.
  Gotcha: deleting the last file in drafts publish/blog/ removes the
  directory → script exits 1; publish/blog/.gitkeep added.
  Obsidian→drafts sync cron exists only in docs — entry point today is a
  manual push to drafts main; recreate the cron when blogging resumes.
  Measured 2026-09-19: Notify Blog green; dispatch run #179 green end to
  end (no changes). Untested: an actual post through the PAT-push step.

- 2026-09-15 — GitHub owner rename + Pages move (commits 4a65290, 9adb6b7,
  e69613d; deploy #3). Found: the newsletter form had been missing on the live
  site since the Actions deploy (form id never passed to CI); fixed by the
  repo variable. Domain move done in the dashboard; Cloudflare rewrote both
  CNAMEs itself. Measured: `neurogenesis.dev` and `www` 200, form present.
  Lesson: a Claude session reported a branch it never created — check
  `git status -sb` before pushing.

## Related

- [[neurogenesis-dev-dns-and-email]] — DNS, routing, Send-as
- `freelance-practice/business/web-presence.md` — copy, gate, Impressum facts (Mentor)
- `freelance-practice/decisions/fdr-004-…` — the decision
- [[../../netcup-vps/decisions/20260911-persistent-credentials-on-netcup-vps|ADR-0034]] — deploy key row
- [[../../netcup-vps/runbooks/automations-stage-2|automations stage 2]] §2.9 — the OAuth client this site's Datenschutz URL serves
