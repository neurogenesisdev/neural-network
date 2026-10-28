---
type: note
status: active
owner: architect
created: 2026-09-27
updated: 2026-09-27
tags: [website, marketing]
---

# neurogenesis.dev — round two (Architect)

> [!NOTE] Summary
> Build tracker for the round-two backlog in [[web-presence]] ("Round two
> backlog", Mentor handoff 2026-09-27). Standing rules from
> [[site-redesign-round-one]] §3.1b apply unchanged (bands, centring, links,
> screenshot loop, Codex gate, merge flow).

## R1 — /about + redirects + articles ✅ 2026-09-27 (branch `site/r1-about`, merged)

Scope: backlog items 1, 2, 3, 4 in one task. Three commits: 1e3d320 (first layout, rejected by luis), 927fd19 (second layout), socials/footer fix. Codex clean on each. Luis on the result: exactly what he imagined.

Final layout (luis, 2026-09-27):
- **Hero (home-hero gradient, content height) holds everything personal:** headshot beside the text column (240 px at ≥ md, 88 px beside name/city on phones), then name, city, What I do (+ Currently), The name, Interests. Headings left-aligned in the text column. Headshot re-cropped tighter (crop 125,168 1180² of the 1440² source); interim — the car interior still shows.
- **Contact** band (navy): heading → email → booking button → reply line, stacked, centred. The page's one booking button.
- **Socials** band (off-white): renamed from "Profiles" (EN + DE), list shrunk to content width and centred, all six linked (LinkedIn, GitHub, Instagram, X, Facebook, Threads — all `neurogenesisdev`).
- **Footer (site-wide):** icons rendered from SOCIAL_LINKS, all six, same order.
- Section headings EN "What I do · The name · Interests · Contact · Socials", DE "Was ich mache · Der Name · Interessen · Kontakt · Socials".
- City line and hero body text white (grey fails contrast on the lower gradient); pairs in check-contrast.mjs.
- `/now`, `/uses`, `/changelog` deleted, 301 → `/about`.
- All four AI-written posts (building-in-public, hello-world, structured-ai-workflow-01-foundation, telegram-notifications-claude-code) `draft: true`, kept as reference; excluded from routes, RSS, sitemap, tags, Pagefind, OG. /articles = "First articles coming soon." + newsletter band. Post URLs 301 → /articles; /tags, /tags/* 302 → /articles (sitemap filter on /tags — remove with the first published post).
- Side fix: newsletter input `min-w-0` — closes the 320 px overflow noted in T7.

## R2 — nav, footer, /services live, /projects, centring (branch `site/r2-nav-footer`, f3b2ee2 + redirect fix, staging pending)

Build notes (2026-09-27): nav links sit in the bar from 1024 px, below that in the menu (Contact always in the bar). /about wrap: with centred section headings the first heading always clears the float (name + city are shorter than the photo), so nothing wraps — Luis picks between the cleared build and the uncleared variant screenshot. Codex P2s: `/projects/memogenesis` and `/de/work/` lost their redirects → fix commit. `/de/` home still renders "Folgt in Kürze." 7× (offer heading, automation band heading/lead/alts/captions) → Mentor.

Luis, 2026-09-27, after clicking through live R1. Structure only — no new copy.
- **/services goes live** (luis): employer consent is in writing and the Gewerbeanmeldung is filed; enquiries before the Gewerbeschein arrives count as preparation/contact, not paid work. `SERVICES_LIVE=true` → Services in nav, indexed, in sitemap. Mentor records it. All /services copy was settled in T5 except the DE heading of the process block ("Folgt in Kürze.") — Architect proposed "So läuft ein Projekt ab" for Luis to approve.
- **/work → /projects** (luis): "Work" collides with the services offered; "Projects" matches DE "Projekte". Routes /projects, /de/projects; the old `/projects → /work` 301s from T6 reversed.
- **Header nav:** Services · Projects · Articles · Newsletter · About, then Contact (accent), EN/DE.
- **Footer (luis, revised):** not the Huberman side column. Three centred columns: Pages (as now) · Updates (Newsletter) · Contact (Contact; FAQ later). Legal (Impressum · Privacy/Datenschutz) as fine print at the bottom, above the divider; below it © left, six social icons right.
- **/newsletter:** main band navy like every other newsletter mention; "← Browse the blog" → "← Browse articles" (arrow kept: Articles sits left of Newsletter in the nav now). Copy itself waits for the Mentor.
- **/about (luis, after screenshots):** wrap tried and rejected (centred heading beside a float looks bad); full-width header block above the text also rejected. Final: R1 two-column layout, but at ≥ lg the text column (608 px) is centred on the page and the 240 px headshot hangs into the left margin; section headings centred in the text column (= page centre, same axis as Contact/Socials); name + city left at the top of the text column. md–lg and phones keep the R1 layouts. Check: text-block centre within 4 px of the Contact heading centre at 1280/1440.
- **Site-wide rule (luis): headings centred.** Exceptions: /about hero sections (above), headings inside article and legal prose.

### R2 status (2026-09-27 15:30) — on staging, NOT merged; held until the design system (R3) exists

Commits f3b2ee2 + a24defc on `site/r2-nav-footer`, Codex clean. Luis's review of staging:
- /about hero headings: centred doesn't work → left-aligned with the text blocks.
- Footer: three centred columns only span a narrow strip above a full-width divider, and the one-item Updates/Contact columns next to the long Pages list look absurd → Huberman layout after all (bold pages list left, divider, category columns right, full container width).
- /articles: heading has no supporting intro line.
- /newsletter: copy not yet settled; intro (heading + text) must be its own band, separate colour from the Kit form band; form module identical everywhere.
- /contact: still old left-aligned layout on a plain black band.
- Home: the contact section is grey.
- Luis: stop patchworking — one design philosophy, applied to every page without error. No more fixes until the Mentor session delivers copy and section rules; then one sweep.

## R3 — design system + full-site sweep (branch `site/r3-sweep`, on top of R2)

1. ✅ Mentor + Luis settled copy and section rules (web-presence.md, 2026-09-27 17:00).
2. ✅ Architect wrote [[site-design-system]] (2026-09-27).
3. S1 sweep: every page against the design system; S2 contact form (Pages Function + Brevo + Turnstile + privacy). Both on `site/r3-sweep`; then Codex, staging, Luis review, merge R2+R3 together (fast-forward).

Architect interpretations awaiting Luis's OK (in the design system): adjacent-band rule = no two adjacent bands of the same colour; band headings centred, headings inside a text column left; contact band colour follows alternation; /about intro carries the What I do paragraph, content band = The name + Interests; no blue text links; DE twins for /articles and /newsletter; proposed 404 + contact-error copy.

## Mentor (copy + design discovery, handoff 2026-09-27)

- /newsletter copy: no articles or newsletters sent yet; what he might write about.
- Page-intro bands: /projects, /articles, /privacy, /impressum each carry a different kind of intro text, /articles has none, /newsletter, /about, /contact have no intro band — decide one pattern and the copy.
- /about interests: add Biohacking (nootropics, health/wellbeing/productivity methods) and maybe one or two more.
- /services live: record the gate decision.
- Footer Contact column: FAQ later.

### R3 status (2026-09-27 23:00)

S1 + S2 + R4 (design system v2) on `site/r3-sweep`, Codex clean, on staging. Luis: "looking pretty damn good" — merge, then R5 polish.

### Live checks after merge (2026-09-27 23:40)

- R2+R3 merged, production live; /services + /de/services linked and indexed; `/work` → `/projects` 301; Turnstile works on production (switch to Invisible mode pending).
- Newsletter: test signup produced no confirmation mail. Kit shows an **account verification hold** (questions answered in Kit chat 2026-09-27, review up to 24 h) — likely blocks confirmation mails. Re-test after Kit approves; check Subscribers with the "Status is Confirmed" filter cleared.
- Contact form: live EN + DE send test still to do.

## R5 — polish after merge (luis, 2026-09-27)

- Home hero: taller (~60–70 vh) so desktop no longer shows three bands at once; the offer is the headline, the name secondary (services business, not a personality brand); offer line larger, possibly extended with the first sentence of the /services expansion.
- Intro gradient: transition starts at ~50–60 % of the height and ends lighter (more grey toward white) — keep text contrast checked.
- /about Socials: centred vs left — Luis to decide.
- /contact: email above the form (Chris Williamson style) vs below with a separator line — decide after seeing the live Turnstile widget (or switch the widget to Invisible mode).
- /about copy: Luis reflecting on self-presentation → Mentor session.
- Hover/animation pass (after layout + content are settled): social icon colour, project/article row highlight (already on the neurogenesis.dev project), nav item hover.
- Every project row gets a link target: automation stack → its public GitHub repo once R4 GitHub cleanup is done (or a case-study page).
- Staging subdomain on neurogenesis.dev so Turnstile works on staging (optional).

## Backlog (luis)

- Language switch as a single button showing only the current language, with a hover/click menu offering the other — cleaner than "EN / DE". Experiment later.

### R5 status (2026-09-28 05:15)

Built on `site/r5` (077c721 … 09cee34): /about R5 copy, layout A with centred intro (photo left, ≥ 800px), all /about headings left; Threads removed; home hero ~65 vh, offer line one step larger; gradient end #707070 from 55 %; /contact email line above the form; result pages on off-white; Turnstile Privacy Addendum in both privacy texts; American spelling. Luis: "that's it" — merge, Turnstile → Invisible, live form test, then the site goes into all social bios. Still open: Kit account verification → newsletter re-test; en-GB date format on EN legal pages (optional switch to US format).

## Round one leftovers (luis)

- T8 demo media ("Automation in action" band still shows two empty tiles), T9 QA + ship, T10 cursor effect — still open; re-scope against design system v2/R5 before starting (next session, 2026-09-28): T9's checklist is largely replaced by check-design.mjs; T10 belongs to the hover/animation pass.

## Later items

- R3 profile audit — Luis (after R2 is live).
- R4 GitHub cleanup — gitleaks full-history scan before any repo goes public.
- R5 writing pipeline + authoring guide — after Luis's first hand-written article. `publish-from-vault.ts` ignores `draft`, and the drafts repo still holds the AI posts without it; a push to its `main` would republish them — fix both before re-enabling vault-publish.
