---
type: note
status: active
owner: architect
created: 2026-09-21
updated: 2026-09-25
tags: [website, marketing, automation]
---

# neurogenesis.dev redesign — round one (Architect)

> [!NOTE] Summary
> Architect's answer to [[site-design-brief]] and
> [[../decisions/fdr-007-bilingual-site-with-language-switch|FDR-007]]:
> design inventory of the live site against the brief, three accent blues to
> pick from, the Claude Code task list (10 tasks, ~14.5 h, shippable after
> every task, cursor effect last), and the demo-loop plan (dedicated demo
> Gmail + fabricated inbox + n8n classifier). Technical routing decisions:
> infra-docs ADR-0038 (`shared/decisions/20260921-neurogenesis-dev-i18n-routing.md`).
> Decided 2026-09-21 (luis): accent **B `#1F6FEB`**; all four ADR-0038 flags yes
> (post URLs move to `/articles/<slug>`); budget 15–20 h counted from
> 2026-09-21, Chantal's site has priority. Waiting on: Mentor's EN copy for
> /services, /contact, offer line (after 2026-09-22).

## 1. Design inventory — live site vs. brief (measured 2026-09-21)

| Area | Today | Brief | Action |
|---|---|---|---|
| Brand top-left | "neurogenesis.dev" | "Luis Gordon" + offer line under it | T2 |
| Nav | Blog · Projects · About, search ⌘K, theme toggle, mobile `details` menu; sticky with **backdrop blur** | right-aligned Services · Work · Articles · About + Contact (accent) + EN/DE; solid, no blur; no search on home | T2; search only on /articles |
| Theme | dark default + `html.light` toggle | fixed bands (dark → light → grey) | remove toggle (flag 1) |
| Colour | purple/violet accent, secondary accent, glow shadows, gradient text | black/white/grey + one blue; screenshots the only other colour | T1 |
| Hero | "Hey, I'm Luis" + personal tagline, gradient text, ScrollReveal | full-height black→grey gradient, name, offer line, booking button, no portrait | T4 |
| Home sections | Recent posts, Projects (MemoGenesis "active"), newsletter (form rendered twice: intro + form heading) | offer → dark media (demo loop, n8n canvas) → process/"how I work" → About teaser → testimonial slot → newsletter → footer | T4; MemoGenesis out |
| Fonts | Inter variable + JetBrains Mono self-hosted, no licence files | Inter + open mono, licences verified | keep both, add OFL files (T1) |
| Motion | View transitions (ClientRouter), ScrollReveal pop-ins | subdued CSS transitions; no pop-ins; one cursor effect at most | remove ClientRouter (flag 2), drop ScrollReveal |
| Pages | /, /blog, /blog/*, /projects, /projects/memogenesis, /about (EN); /impressum, /datenschutz, /contact, /services (DE, noindex); /privacy (EN); /newsletter, /changelog, /tags | EN at `/`: home, services, work, articles, about, contact, impressum, privacy; DE under `/de/`: home, services, work, about, contact, impressum, datenschutz | T3, T5–T7 |
| Legal | DE canonical, generator text 15.09.2026, processors: Cloudflare, Google, Kit, Cal.com, Giscus, GitHub, jsDelivr, X, LinkedIn, YouTube — **no Brevo** | + Brevo in DE and EN; EN marked convenience translation | T7 |
| /services | DE, `noindex,nofollow`, unlinked, sitemap-filtered, Cal.com button | same gate in both languages; standalone landing with mini-hero | T5 |
| Contact | obfuscated mailto + Cal.com button | always one click away (nav accent) | T2 |
| Footer | © + socials + Newsletter · Changelog · Impressum · Datenschutz · Kontakt | grey band, legal footer-only, socials, per-language labels | T2 |
| Media | none | dark-mode screenshots/loop, ≤ 2 MB, reduced-motion fallback | T7b ✅ 2026-09-25 | Giscus removed (Luis: a business site needs no comments) — merged (63ee047). Component, config, CSS, CSP (`giscus.app`) gone; Giscus bullet + generator "Kommentare und Beiträge" module removed from datenschutz.md, Giscus paragraph from privacy.astro, Stand/Last updated 2026-09-25; GitHub no longer a DE processor (EN keeps the build-time Gist note). **For T9:** Blogs section boilerplate still lists "Inhaltsdaten … Beiträge" and "Feedback" (reader content no longer accepted) — prune in a legal pass; GitHub Discussions off in repo settings (Luis) | 0.25 | T7 | |
| T8 |
| Meta | description "Personal blog and project showcase… biohacking" | business first | T4 (per-language descriptions) |

Everything in the brief's *Non-goals* is untouched: Astro/Pages, no tracking
banner, fonts self-hosted, no CMS, /services gate.

## 2. Accent blue — pick one (luis)

All three sit at hue 215–221 (no violet cast). Ratios are WCAG contrast; ≥ 4.5 passes AA for text, ≥ 3.0 for UI edges.

| | Hex | White text on it | Edge vs. black hero (#0a0a0a) | As link text on white | Character |
|---|---|---|---|---|---|
| **A** | `#0A6CFF` | 4.56 ✓ | 4.34 ✓ | 4.56 ✓ | electric, Linear-like; the loudest of the three |
| **B** | `#1F6FEB` | 4.63 ✓ | 4.27 ✓ | 4.63 ✓ | GitHub-dark blue; tech-familiar, calm |
| **C** | `#2563EB` | 5.17 ✓ | 3.83 ✓ | 5.17 ✓ | deeper royal blue; most conventional, slightly warmer |

**Chosen (luis, 2026-09-21): B `#1F6FEB`**; hover `#1A63D6`. Usage rule: the
blue appears only on the booking/contact action (button fill or a Snellenberg
circle behind the nav "Contact"), plus focus rings. Hover = same hue, 8 %
darker. Nothing else on the page is coloured except screenshots.

## 3. Claude Code task list (budget 15–20 h from 2026-09-21, log as `h <hours> site <note>`)

Rules: one task per branch `site/tN-slug` → `/codex:review` → Luis merges →
staging deploy (`workflow_dispatch`, "Use workflow from" = the branch) → phone
check → merge to `main`. After every deploy that touches the build: newsletter
form check on the live URL the same day (Mentor standard). **Every task ends
in a deployable site** — cut-off at any T leaves the previous T live.

Budget (luis, 2026-09-21): hours count from today; 15–20 h; the 3 h `site`
of 2026-09-17 are pre-brief. Luis's stance: the Gewerbe isn't approved yet
and the business is still being built, so the working-hours argument doesn't
bind him now — he spends what he wants as long as Chantal's site is not
delayed. Mentor's 15 h cap stands as the point where round one is *declared*
shipped and leftovers move to the backlog; T10 and polish may use the 15–20 h
band.

| # | Task | Est. | Depends | Done when |
|---|---|---|---|---|
| T0 | Decisions (luis): blue, four flags (ADR-0038), branch created | 0.5 | — | this note updated with the picks |
| T1 ✅ 2026-09-22 | Tokens, fonts, licences — live (8049023, 223d7e5, 4cef069; Codex P2 on link contrast → `--color-accent-text #6CA8FF` for text on dark, white-on-accent buttons, `scripts/check-contrast.mjs` in `npm run lint`) | 1.0 | T0 | greyscale + one accent token; purple/glow/gradient utilities and `html.light` gone; OFL files in `public/fonts/`; build green; site live recoloured |
| T2 ✅ 2026-09-22 | Layout shell — merged (8fcaaad…433c270). Also done here, pulled forward by Codex: `/blog` → `/articles` and `/projects` → `/work` index routes + redirects (detail pages still `/blog/<slug>`, `/projects/<slug>` — T7/T6 move them), SearchButton on /articles + posts, `isActivePath` in constants, `paper-2` token, footer dark, offer line hidden < sm, `astro:page-load` shim in BaseLayout (delete when ObfuscatedEmail is migrated, T3). DE nav hrefs point at EN routes with a TODO until T3. DE labels placeholder: Leistungen · Projekte · Artikel · Über mich · Kontakt — Mentor confirms with the offer line. Search modal still global; moves to /articles in T7 | 1.5 | T1 | name + offer line, right nav, solid sticky, Contact accent, switch placeholder, grey footer; ClientRouter removed |
| T3 ✅ 2026-09-23 | i18n routing — merged (site/t3-i18n, 4d05055). Built-in i18n, six shared page components under `src/components/pages/`, DE stubs for home/about/work ("Diese Seite folgt in Kürze.", TODO(T4)/(T6)), `LangSwitch` derives the counterpart from each page's `alternates`/`switchTo` (unpaired pages → `/de/` with the German hint), sitemap `serialize` injects the privacy↔datenschutz pair, `check-i18n-twins.mjs` in lint, `/datenschutz` → `/de/datenschutz` 301, ObfuscatedEmail shim gone. Codex P2s fixed: no `/projects/*` splat until T6; 404 switch → `/de/`. Open: Google OAuth consent-screen privacy URL → `/de/datenschutz` (Mentor/Luis, 301 covers it) | 2.0 | T2 | `/de/` twins for home/about/contact/impressum/datenschutz with existing DE copy, `LangSwitch` works both ways, `_redirects` live, hreflang + sitemap alternates match |
| T4 ✅ 2026-09-23 | Home — merged (site/t4-home, c7c7ad0 + 3 hero passes). Six bands, `Home.astro` shared component, DE with Mentor-approved German only + `Folgt in Kürze.` placeholders, newsletter EN-only, `--color-bg-3 #555`, eased hero fade, `references/linear-hero-1280.png`. **Luis, 2026-09-23: gradient still short of the Linear reference and the hero→white transition glares** — parked: T8 adds the real dark media + a radial halo behind the demo tile and moves the offer band to `surface-1`; further gradient work after the sandbox lets CC see its own screenshots (§4a) | 2.0 | T3 | hero gradient band, offer line (placeholder text until Mentor copy), booking button, offer cards, media slot (poster placeholder), process strip in the testimonial slot, About teaser, newsletter; passes phone check |
| T5 ✅ 2026-09-24 (merge pending phone check) | /services EN + DE — d71271d + visual pass 57fc909 + email-link contrast fix + header fits 320px; Codex clean on the third round. First task with the in-agent screenshot loop (sandbox off). Visual pass fixed: wordmark read as a second button (now small tag-style caps, 2px radius), H1 leading, stretched cards (now top-aligned, uneven heights accepted), price line as a hairline note not a banner. Codex caught what screenshots didn't: `accent-text` email link on the light contact band (2.2:1) → `ObfuscatedEmail tone=light` = ink-0 underline; wordmark overlapping Contact at 320–360px. Rule confirmed: **links on light bands = ink-0 underline; accent-text is dark-band-only.** DE process heading on home and /services still `Folgt in Kürze.` (no settled DE heading — Mentor). | /services EN + DE standalone landing — **copy settled 2026-09-24** (web-presence.md §"/services copy — T5"); scope grew: header wordmark `NEUROGENESIS DEV` in a blue outline replaces the name top-left (Luis/Mentor decision — note: second accent use vs. the brief's one-accent rule, accepted), offer line + button label change site-wide via ui.ts, home process block → 5 steps. Prompt in chat 2026-09-24 (supersedes the §3.2 T5 block). /about redesign parked for its own session (web-presence.md "Parked: /about") | 2.0 | T3, Mentor copy | mini-hero, offers, process, About block, booking; `SERVICES_LIVE=false` in `constants.ts` drives noindex + sitemap filter + hidden nav item in both languages |
| T5b visual reset ✅ 2026-09-24 merged with T5 (fd9e7c9 + home cards/about fix) — Luis: "much better, no longer want to restart the site" | Luis rejected the T5 look. **Architect's error:** the T5 prompt added an offer line under the logo, rounded corners and white logo text — none of it was Luis's spec ("rectangular box, blue, logo only"). Rule: implement Luis's spec literally first; Architect opinions go in a separate line. Luis's spec, overriding the brief: Huberman logo (square box, 2px #1F6FEB border, blue text, wide tracking, nothing under it); **distinct colour per band, never two light bands adjacent** (navy #16263B, off-white #EEEAE6, white, light grey #E4E4E4, charcoal footer #2B2B2B — replaces "greyscale + one blue", colour-coded bands pulled forward from round two); centred headings/leads/buttons/grids; **one booking button per page** (hero), contact block = email under heading; Contact link header-only; footer = Huberman columns + divider + copyright/socials row; price line plain centred; Automation card shortened to match the others (EN/DE text in the prompt, supersedes web-presence card 2). References in repo `references/` (huberman-logo, huberman-footer, chriswillx-bands, chriswillx-footer). Luis considered a rebuild from scratch; Architect advised a presentation-layer reset because routing/i18n/legal/deploy are sound | — | T5 | |
| T6 ✅ 2026-09-24 | /work timeline — merged (f38e60e + scoping fix). Chiang list (140 px year column, 608 px body, measured vs `references/chiang-work-1280.png`), schema title/summary/role `{en, de}`, entries automation-stack + neurogenesis-dev, memogenesis unpublished, `/projects/[slug]` + ProjectLayout/ProjectCard deleted, `/projects/* /work 301`. Codex P2 caught what an 8-page pixel diff missed: `flex-col` on the shared `<main>` shrank `mx-auto` stubs (/de/about) → scoped via BaseLayout `fillMain` prop (Work only). Lesson: regression diffs must cover every built page (`cmp` over dist), not a sample | 1.5 | T3 | `content/projects` repurposed: this site + automation stack entries with tech tags; memogenesis `published: false`; `/projects*` → `/work` |
| T7 ✅ 2026-09-25 | Articles + legal restyle, Brevo — merged (251316c + callout fix). Posts at `/articles/<slug>`, `/blog/series/*` and `/blog/*` 301s, search (Pagefind) only on /articles + posts, 404 "Browse articles" link, navy intro → white/off-white band on posts, listings and legal pages, `prose-ink` light prose (code blocks dark), shared `NewsletterBand` (navy, home + posts). Brevo entry in datenschutz.md (Kontakt- und Anfrageverwaltung, after Gmail; Brevo GmbH Berlin = contracting entity for DE per Brevo ToS) + EN paragraph in privacy.astro, Stand 2026-09-24. Codex P2 caught callouts keeping dark prose on white (screenshots missed: hello-world has no callouts). Giscus thread-identity P2 dismissed (ADR-0038) → Luis: remove Giscus entirely (T7b, running). **For T9:** NewsletterForm plain overflows at 320 px (Subscribe clipped, also on home); still old dark layout: /about, /de/about, /contact, /de/contact, /tags, /tags/*, /404, /changelog, /newsletter, /now, /uses | 1.5 | T2 | `/blog*` → `/articles*` (index and posts, 301s), light reading theme, legal pages in new shell, convenience line on EN, Brevo in both privacy files, search only on /articles |
| T8 | Demo media embed (recording is Luis's, §4) | 0.5 | T4, §4 done | `<video muted loop playsinline autoplay>` webm+mp4 ≤ 2 MB + poster; reduced-motion → poster; n8n canvas as AVIF/WebP with 2× source; on home and both /services |
| T9 | QA + ship | 1.0 | all | checklist in §3.1 green; merged; live; form tested; hours logged |
| T10 | One first-screen effect showing JS/CSS craft — **last, in the 15–20 h band** | 1.0 | T9 | candidate: subdued glow following the pointer; Luis may pick another confined effect (Comeau-style dose). Static on touch and `prefers-reduced-motion`, no layout shift, no dependency |

Total T0–T9: **13.0 h** (+ ~1.5 h of Luis's recording time in §4 = 14.5). T10 fits only if T1–T9 come in under estimate.

### 3.1 QA checklist (T9)

```
# from the repo root, after `astro build`
grep -rlE 'Kunden\b|Agentur|erfahrungsgemäß|\bclients\b|\bagency\b' dist/ --include=*.html   # must print nothing
grep -c 'hreflang="x-default"' dist/index.html dist/de/index.html                            # 1 and 1
grep -o 'hreflang="[a-z-]*"' dist/services/index.html | sort | uniq -c                        # en, de, x-default
grep -c 'name="robots" content="noindex' dist/services/index.html dist/de/services/index.html  # 1 and 1
grep -c 'services' dist/sitemap-0.xml                                                          # 0
grep -c 'data-form-id' dist/index.html dist/de/index.html                                      # ≥1 each
grep -rl 'backdrop-blur\|ClientRouter\|html.light' src/                                        # nothing
find public/fonts -name 'OFL-*.txt' | wc -l                                                    # 2
```

Then on the phone (staging URL): offer readable and booking tappable without
scrolling; switch EN↔DE on every business page; `/datenschutz` 301s to
`/de/datenschutz`; Lighthouse ≥ 90 performance/accessibility on home.

### 3.1b Standing rules for T6–T9 (from T5 + visual reset, 2026-09-24)

**Design system (Luis's spec, overrides the brief):**
- Header: Huberman logo box only (square corners, 2px #1F6FEB border, blue caps, 0.3em tracking), nothing under it; right: nav, Contact, EN/DE (switch in the mobile menu < sm). Must fit 320px.
- Bands: every section its own colour, never two light bands adjacent — navy #16263B, off-white #EEEAE6, white, light grey #E4E4E4; hero black→grey gradient; footer charcoal #2B2B2B with Huberman columns (Pages · Updates · Legal) + divider + copyright left / socials right.
- Centred headings, leads, buttons and card grids; card body text may stay left-aligned. Cards equal height.
- One booking button per page, in the hero. Contact link only in the header. Email shown as a centred link under a heading, never next to a button.
- Links: dark bands `accent-text`; light bands `ink-0` underline. Every new text/background pair goes into `scripts/check-contrast.mjs`.
- Shared copy has one source (cards, About) so pages can't drift.
- References live in the repo `references/` (huberman-logo, huberman-footer, chriswillx-bands, chriswillx-footer, chriswillx-full, linear-hero-1280). Prompts cite a reference file + measured values, never adjectives.

**Claude Code workflow (proved on T5):**
- Sandbox off; browser MCP (Playwright) and `scripts/shot.sh` both work. Every visual task: build → serve dist/ on :4173 → screenshot 320/390/1280 (full page) → Read → fix → repeat; the agent judges against the references before reporting. Use the frontend-design skill.
- Fresh session per task; plan mode for multi-file tasks; foreground only.
- Codex reviews the diff only (no screenshots) — it catches contrast/overlap bugs the screenshots miss; both gates stay.
- Implement Luis's spec literally first; any Architect/agent opinion goes in a separate line, never into the build.
- Luis looks at `.shots/` via `scp -r vps-dev:dev/neurogenesis-dev/.shots ~/Downloads/shots` (modern scp doesn't expand `{}` remotely); the Architect reads them from Downloads.
- Merge flow: push branch → staging dispatch ("Use workflow from" + branch input = the task branch) → phone → `merge --ff-only` → delete branch → form check → hours.

### 3.2 Prompts (paste as-is, one per task, in the repo on netcup-vps as `dev`)

> T6–T9 prompts below predate 3.1b — prepend: "Follow §3.1b of ~/dev/freelance-practice/business/site-redesign-round-one.md (design system + visual loop)."

Prefix every prompt with: **"Run this in the foreground yourself; do not
delegate to a background agent."** Lesson 2026-09-21: T1 was handed to the
`builder` subagent in the background; its file-delete approval prompt never
surfaced in the main pane and the agent sat paused for 9 h while Luis was
away. Background agents need someone present to approve; these tasks don't
need a subagent at all. Claude Code reads the vault via the symlink `~/dev/freelance-practice →
/srv/vaults/freelance-practice` (Read tool works; Bash in the sandbox does
not — `/srv` is in the built-in deny list and `sandbox.filesystem.allowRead`
is ignored in 2.1.278). Use that path in prompts. Rule for every prompt:
**style and taste from the brief; copy only from web-presence.md or the
Mentor — nothing else in the vault is source text.**

**T1**

```
Context: neurogenesis.dev, Astro 6.4 + Tailwind v4, tokens in src/styles/global.css. Redesign round one (freelance-practice/business/site-design-brief.md): greyscale + one blue accent, never purple.
Task: (1) Replace the colour tokens with a greyscale scale (bg-0 #0a0a0a, bg-1 #141414, bg-2 #1f1f1f for dark bands; surface #ffffff, surface-2 #f5f5f5, surface-3 #e5e5e5 for light bands; text tokens for each) plus exactly one accent token `--color-accent: <HEX FROM LUIS>` and `--color-accent-hover` (8% darker). (2) Remove accent-secondary, glow, gradient-text utilities and the `html.light` token block and theme toggle component; delete their usages (build must stay green). (3) Add public/fonts/OFL-Inter.txt and public/fonts/OFL-JetBrainsMono.txt with the upstream SIL OFL 1.1 texts (rsms/inter LICENSE.txt, JetBrains/JetBrainsMono OFL.txt) and a public/fonts/README.md listing font, version, licence.
Do not: change page structure, copy, routes, or dependencies. Do not touch content/.
Verify: node --import tsx scripts (as in CLAUDE.md), astro build, npm run check (only the known rehype-sidenotes error), npm run lint; `grep -rn 'violet\|purple\|glow\|gradient-text\|html.light' src/` prints nothing.
Branch site/t1-tokens. Commit. Report the diff summary.
```

**T2**

```
Run this in the foreground yourself; do not delegate to a background agent.
Context: same repo, branch from main after T1. Tokens now: bg-0/1/2, surface-0/1/2, ink-0/1/2, paper-0/1, accent #1F6FEB, accent-hover, accent-text #6CA8FF (text on dark). Add --color-paper-2: #737373 as the muted-text-on-dark token and move dark-band muted text from ink-2 to paper-2 (ink-* is for light bands only); scripts/check-contrast.mjs gains the pair paper-2/bg-0 (large/secondary text, threshold 3.0). Brief: header shows "Luis Gordon" top-left with the offer line under it (offer text from src/i18n/ui.ts, placeholder "Websites and automations for independent professionals" / DE "Websites und Automatisierungen für Selbstständige" until the Mentor delivers copy); nav right-aligned: Services · Work · Articles · About, then Contact styled as the single accent action, then an EN/DE switch slot; sticky, solid background, no backdrop blur; mobile menu keeps the details/summary pattern.
Task: (1) Rewrite src/components/Header.astro and Footer.astro accordingly; footer is a grey band with © Luis Gordon, socials, Newsletter · Impressum · Privacy/Datenschutz · Contact/Kontakt via ui.ts labels. (2) Add src/components/Band.astro with a `tone` prop (dark | light | muted) that sets background/text tokens and full-width padding. (3) BaseLayout: accept `lang` (default 'en') for <html lang>, and `alternates?: {en: string; de: string}`; when given, emit rel=alternate hreflang en/de/x-default (absolute URLs from Astro.site; x-default = en). (4) Remove ClientRouter/view transitions and the ScrollReveal component and its usages; keep CSS transitions ≤ 200 ms. (5) Create src/i18n/ui.ts with `en` and `de` objects for nav, footer, buttons, switch. Services nav item is rendered only when SERVICES_LIVE (add to src/lib/constants.ts, false).
Do not: add i18n routing yet, change page copy, or add dependencies.
Verify: build gate as in CLAUDE.md; `grep -rn 'backdrop-blur\|ClientRouter\|ScrollReveal' src/` prints nothing; header renders on / and /blog.
Branch site/t2-shell. Commit. Report the diff summary.
```

**T3**

```
Run this in the foreground yourself; do not delegate to a background agent.
Context: same repo. Decision record: ~/dev/infra-docs/shared/decisions/20260921-neurogenesis-dev-i18n-routing.md (ADR-0038) — read it first. Brief: ~/dev/freelance-practice/business/site-design-brief.md. Copy source: ~/dev/freelance-practice/business/web-presence.md only. EN at /, DE under /de/, no auto-redirect, articles EN-only. Posts move to /articles/<slug> in T7, not here. T2 left: DE nav hrefs on EN routes with a TODO (fix here); an astro:page-load shim in BaseLayout for ObfuscatedEmail (migrate the component to DOMContentLoaded/module init and delete the shim).
Task: (1) astro.config.mjs: i18n { defaultLocale 'en', locales ['en','de'], routing { prefixDefaultLocale: false } }; sitemap i18n option per the ADR; sitemap filter excludes /services and /de/services. (2) Move DE pages: /datenschutz → src/pages/de/datenschutz.astro, /contact → de/contact.astro, /impressum → de/impressum.astro, /services → de/services.astro (keep their DE copy and the noindex meta on services); create EN twins at the old paths with the existing EN texts where they exist (/privacy stays; /impressum EN = same facts, English labels; /contact EN and /services EN use TODO-marked placeholder copy — Mentor delivers). Each page pair renders one shared component under src/components/pages/ with a `lang` prop and a copy object; pass `alternates` to BaseLayout. (3) src/components/LangSwitch.astro: derives the counterpart URL (/x ↔ /de/x); on EN-only routes (/articles, /blog/*, /tags/*, /changelog, /newsletter) links to /de/ with title="Artikel gibt es nur auf Englisch"; accepts a `switchTo` override. Mount it in Header. (4) public/_redirects: `/datenschutz /de/datenschutz 301`, `/blog /articles 301`, `/projects /work 301` (no `/projects/*` splat — detail pages still exist; T6 adds it. `/blog/*` splat comes with T7). (5) Kit form privacy link: /privacy on EN pages, /de/datenschutz on DE pages. (6) scripts/check-i18n-twins.mjs: fails if a file in src/pages/de/ has no twin in src/pages/ or vice versa (allowlist EN-only routes); wire into `npm run lint`.
Do not: touch src/pages/blog/, add i18n libraries, add fallback locales, redirect by browser language.
Verify: build gate; `grep -o 'hreflang="[a-z-]*"' dist/contact/index.html | sort -u` shows de, en, x-default; dist/de/contact/index.html exists; dist/sitemap-0.xml has xhtml:link entries and no services URL; node scripts/check-i18n-twins.mjs exits 0.
Branch site/t3-i18n. Commit. Report the diff summary and the list of TODO placeholders.
```

**T4**

```
Context: same repo, after T3. Brief sections for home: (1) full-height first screen, black→grey gradient, "Luis Gordon", offer line, one booking button (BOOKING_URL, accent), no portrait; (2) light band: offer in three short cards (Website · Automation · Care plan; DE Website · Automatisierung · Website-Pflege) from the services catalog wording in freelance-practice/business/web-presence.md, no prices, no plural clients/Kunden, no agency/Agentur, no claimed experience; (3) light band "Automation, shown": a dark media tile (16:9 poster placeholder public/media/demo-inbox-poster.png, dark #141414 with a small caption) and a second tile for the n8n canvas; (4) muted band "How a project runs": 4 steps (free 30-min call → fixed quote → build with weekly check-in → handover + optional care) — this occupies the future testimonial slot so it never reads empty; (5) light band About teaser: two sentences from web-presence /about appendix + link; (6) newsletter band (single heading, one form — today the heading renders twice, fix); footer.
Task: implement src/components/pages/Home.astro rendered by src/pages/index.astro (en) and src/pages/de/index.astro (de) with copy objects in each page file; meta description per language (business first, no "biohacking"); remove Recent posts and Projects sections and any MemoGenesis reference from home.
Do not: add JS beyond what exists, add motion, add colour besides the accent, touch /blog pages.
Verify: build gate; phone viewport 390×844 in a headless Chromium screenshot (playwright is available on the VPS? if not, skip and say so): offer line and booking button visible without scrolling; `grep -c 'Stay in the loop' dist/index.html` is 1.
Branch site/t4-home. Commit. Report the diff summary.
```

**T5**

```
Context: same repo, after T4 and after the Mentor's EN copy for /services lands in freelance-practice/business/web-presence.md (if not there yet, keep TODO placeholders and say so).
Task: /services and /de/services as standalone landing pages sharing src/components/pages/Services.astro: mini-hero (dark band, offer line, booking button), offers (same three cards as home, one paragraph each), process (4 steps), About block ("Über mich"/"About me" copy from web-presence verbatim), media tile (demo loop slot), booking + obfuscated email. Gate: SERVICES_LIVE=false keeps `<meta name="robots" content="noindex, nofollow">` on both pages, the sitemap filter, and the hidden nav item; document in CLAUDE.md that go-live = flip SERVICES_LIVE and remove the filter.
Do not: publish prices, link the page from anywhere while SERVICES_LIVE is false, alter the DE copy's wording.
Verify: build gate; `grep -c noindex dist/services/index.html dist/de/services/index.html` = 1 and 1; `grep -c services dist/sitemap-0.xml` = 0; `grep -rl '/services' dist/index.html dist/de/index.html` prints nothing.
Branch site/t5-services. Commit. Report the diff summary.
```

**T6**

```
Context: same repo, after T3. Brief: /work = timeline/list of projects with tech tags per entry (brittanychiang.com model, not a résumé). Content today: content/projects/ with memogenesis.
Task: (1) Extend the projects schema if needed (year, role, tags[], links[], published, lang-specific title/summary as {en, de}). (2) Entries: neurogenesis-dev (this site: Astro, Tailwind, Cloudflare Pages, GitHub Actions, bilingual), automation-stack (n8n Gmail classifier, Claude API, Telegram secretary — describe what runs, no client mention); memogenesis → published: false. (3) src/pages/work.astro + de/work.astro via src/components/pages/Work.astro: reverse-chronological list, tag pills reuse TagList (non-linkable), a slot per entry for a dark media tile. Remove /projects/[slug] pages and add `/projects/* /work 301` to public/_redirects in the same commit (T3 deliberately left the splat out while the pages existed).
Do not: mention Chantal or any client; claim experience; add a search.
Verify: build gate; dist/work/index.html and dist/de/work/index.html exist; `grep -ci memogenesis dist/work/index.html` = 0.
Branch site/t6-work. Commit. Report the diff summary.
```

**T7**

```
Context: same repo, after T2. Brief: "Blog" → "Articles"; articles EN-only, restyled not reorganised; legal pages restyled; EN legal pages are convenience translations; Brevo is a new processor.
Task: (1) Move the rest of src/pages/blog/ ([slug] and series/[series]) → src/pages/articles/ (index and pagination moved in T2); add `/blog/series/* /articles/series/:splat 301`; update every internal link, RSS item URLs (feed stays at /rss.xml), OG/canonical, Giscus mapping (accept that old comment threads are orphaned — decided), and add `/blog/* /articles/:splat 301` to public/_redirects; nav label Articles; Pagefind search button only on /articles and post pages. (2) PostLayout and legal pages on the light band with dark code blocks; keep TOC, footnotes, Giscus. (3) Legal: /privacy and /impressum (EN) get a first paragraph "Convenience translation — the German version at /de/datenschutz (resp. /de/impressum) is legally binding." / DE pages unchanged in substance. (4) Add the Brevo section to src/content/legal/datenschutz.md and the EN privacy source — Luis pastes the generator's Brevo module text into the DE file first (see ADR-0038); translate it for EN; set both `updated` to today.
Do not: change post slugs, delete posts, alter the generator's other sections, drop processors.
Verify: build gate; dist/articles/index.html and dist/articles/<some-slug>/index.html exist, dist/blog/ does not; `grep -rl '/blog/' dist/ --include=*.html --include=*.xml` prints nothing; `grep -c Brevo dist/de/datenschutz/index.html dist/privacy/index.html` ≥ 1 each; `grep -c pagefind dist/index.html` = 0.
Branch site/t7-articles-legal. Commit. Report the diff summary.
```

**T8**

```
Context: same repo, after T4. Media produced per freelance-practice/business/site-redesign-round-one.md §4 is in public/media/: demo-inbox.webm, demo-inbox.mp4, demo-inbox-poster.png, n8n-canvas.avif, n8n-canvas.webp, n8n-canvas.png.
Also: offer band background surface-0 → surface-1 (#f5f5f5) and a soft radial halo (bg-3 → transparent, ~60% width) behind the demo tile in the automation band, so the light sections stop glaring next to the dark hero (Luis, 2026-09-23). Task: src/components/DemoLoop.astro: <video autoplay muted loop playsinline preload="metadata" poster> with webm then mp4 sources; when `prefers-reduced-motion: reduce` show the poster image only (CSS-hide video + noscript-safe img fallback); lazy below the fold. src/components/MediaTile.astro for the n8n <picture> (avif, webp, png) with width/height set. Mount both on Home and Services (EN+DE), captions from ui.ts. Assert each video ≤ 2 MB in a lint script.
Do not: autoplay with sound, load media from third parties, add JS.
Verify: build gate; `du -h public/media/*`; Lighthouse performance on / ≥ 90 (use `npx lighthouse` if present, else say so).
Branch site/t8-media. Commit. Report the diff summary.
```

**T9** — run §3.1, fix findings, merge, deploy, test the newsletter form live, `h <hours> site "round one shipped"`.

**T10** (only with hours left)

```
Context: same repo, after T9. One cursor-reactive effect on the first screen only: a soft radial glow (accent at 8% alpha, 600px) following the pointer over the hero band.
Task: ≤ 40 lines vanilla JS in the Hero component, pointer events only (no touch), disabled under prefers-reduced-motion and when (hover: none); CSS custom properties for position; no layout shift, no dependency.
Verify: build gate; Lighthouse performance unchanged (±2); works with JS disabled (static gradient).
Branch site/t10-cursor. Commit. Report the diff summary.
```

## 4. Demo loop — dedicated demo Gmail + fabricated inbox (Luis, ~1.5 h)

Goal per brief: one silent loop of an inbox sorting itself + one dark-mode
n8n canvas screenshot. Fabricated mail, no real people, no real companies, no
implied clients. Reusable for client demos.

**4.1 Account (15 min).** New personal Google account, e.g.
`neurogenesis.demo@gmail.com` (recovery: the hub Gmail; KeePassXC entry
`gmail-demo`). Gmail theme **Dark**, density Compact, hide Chat/Meet panes,
language English. In Google Cloud project `n8n-neurogenesis` → OAuth consent
screen → **Test users** → add the demo address (the app is unverified; only
test users can authorise). No Workspace, cost 0.

**4.2 Seed the inbox (30 min).** Use the Gmail API `messages.import` — it
places a raw RFC 822 message into the mailbox *without sending*, so From
headers can be fabricated. In n8n: new workflow `demo-seed`, one HTTP Request
node with the new Gmail OAuth2 credential (`gmail-demo`), loop over a JSON
list of ~25 messages, `POST https://gmail.googleapis.com/gmail/v1/users/me/messages/import?internalDateSource=dateHeader`
with body `{"raw": "<base64url RFC822>", "labelIds": ["INBOX","UNREAD"]}`.
Fabrication rules: senders only on reserved domains (`example.com`,
`example.org`, `*.test`); invented names that read as invented ("Nord & Fels
Buchhaltung", "Ridgeline Webhosting"); mail types: invoice, hosting receipt,
newsletter, platform security notice, calendar invite, delivery notification,
generic promo, one contact-form-style enquiry (fictional person, no company
name). Dates spread over two weeks so the inbox looks lived-in. Store the
seed JSON in the inbox repo (`freelance-automation-inbox/demo/seed.json`)
for reuse.

**4.3 Classifier on the demo account (15 min).** Duplicate the Gmail
classifier workflow → `gmail-classifier-demo`, swap the credential to
`gmail-demo`, disable the schedule (manual trigger only), keep the same label
set (Actions/*, Topics/*, System/Processed). First run creates the labels.
Anthropic key: the existing `netcup-vps-n8n` workspace; ~25 mails ≈ cents.

**4.4 Record (30 min).** Laptop, Hyprland: `wf-recorder -g "$(slurp)" -f
demo-inbox-raw.mp4` on a 1280×800 region of the dark Gmail inbox (no browser
chrome). Take 1: inbox unlabelled, 2 s still. Run the workflow, refresh, take
2: labels present, 2 s still. Loop with a cross-fade so the last frame returns
to the first:

```
ffmpeg -i demo-inbox-raw.mp4 -vf "fps=24,scale=1280:-2" -an -c:v libvpx-vp9 -b:v 0 -crf 34 public/media/demo-inbox.webm
ffmpeg -i demo-inbox-raw.mp4 -vf "fps=24,scale=1280:-2" -an -c:v libx264 -crf 26 -pix_fmt yuv420p -movflags +faststart public/media/demo-inbox.mp4
ffmpeg -i demo-inbox-raw.mp4 -vf "select=eq(n\,0)" -frames:v 1 public/media/demo-inbox-poster.png
```

Target 8–12 s, ≤ 2 MB each. n8n: Settings → Theme Dark, zoom-to-fit the
classifier canvas, credential names generic, 2× screenshot (`grim` on a
2560-wide region or browser device-pixel-ratio 2) → `n8n-canvas.png`, then
`cwebp -q 82` and `avifenc` (or `ffmpeg -c:v libaom-av1`) for the two web
formats. Check the screenshot for anything real (addresses, names, keys)
before committing.

Fictional enquiry (luis, 2026-09-21): include it — it shows what a client
would get; a real client's inbox could never be shown uncensored, which is the
point of the demo account. Keep the sender obviously fictional (reserved
domain, no company name).

## 4a. Rig follow-ups surfaced by this round (do after T4, not mid-task)

**2026-09-24 — sandbox OFF (luis):** Claude Code's Bash sandbox on netcup-vps `dev` disabled (`/sandbox` → No Sandbox) after it broke Playwright/DevTools MCPs and headless Chromium for the second day. Remaining fences: unprivileged `dev`, per-repo deploy keys, nftables skuid egress rules, branch + Codex review + manual merge. Browser MCPs need `/usr/bin/chromium` overrides (Playwright `--executable-path`, DevTools `--executablePath` + `--chromeArg=--no-sandbox`); registered at user scope with `claude mcp add` so plugin updates don't reset them; `shot.sh` kept as the simple path. Three-way screenshot test passed 2026-09-24 11:40. Runbook entry due in infra-docs `netcup-vps/runbooks/coding-rig-stage-1.md`.

**Done 2026-09-23 (rig session):** `scripts/shot.sh <path|url> <WxH> [out]` in the site repo, listed in the untracked `.claude/settings.json` as `sandbox.excludedCommands` (`scripts/shot.sh *`, `bash scripts/shot.sh *`, `./scripts/shot.sh *`) — measured: Claude Code takes and Reads its own screenshots; output under `.shots/` (gitignored). Plugins installed: `frontend-design`, `playwright`, `chrome-devtools-mcp` (Playwright's own browser still runs inside the sandbox — untested; shot.sh is the known-good path). `opus` alias resolves to `claude-opus-5-5` (checked) → reviewer already on 5.5; builder stays Sonnet until one task is compared. Laptop `vault-push` now loops freelance-practice + infra-docs. Every layout prompt from T5 on ends with: "run scripts/shot.sh at 390x844 and 1280x800, Read both, fix mismatches against the reference, then report."

- Headless Chromium fails inside the Claude Code sandbox (singleton socket). Test from a plain `dev` shell with `--no-sandbox --disable-dev-shm-usage`; if that works, either an `excludedCommands` entry for a `scripts/screenshot.mjs` wrapper or a repo-local user-data-dir. Then install the Playwright, Chrome and `frontend-design@claude-plugins-official` plugins for Claude Code (the last one suggested by CC itself during T4). Luis asked to be reminded (2026-09-23).
- Timer inventory (2026-09-23): laptop `vault-push.timer` = freelance-practice only; VPS `vault-mirror.timer` (15 min) pulls the root mirrors `/srv/vaults/*` (MCP + symlink readers). `dev`'s own clones (`~/dev/infra-docs`, `~/dev/neurogenesis-dev`) are pulled by hand on purpose — no timer should move a working tree under a running agent. To do: laptop push timer covers infra-docs (+ flashcard vault if still edited).
- **Visual specs, not adjectives** (luis, 2026-09-23): three gradient passes on the hero because "Linear-style" is not a spec. Keep a `references/` folder in the site repo with cropped reference screenshots (Linear hero, Snellenberg nav, Chiang timeline…) and write prompts as "match references/x.png: fade starts at 45%, ends #4a4a4a". Claude Code reads PNGs directly. For later rounds: screenshot-to-code tools are an option, but a reference image + measured numbers in the prompt gets most of the value for free.
- `.claude/settings.json` in the repo: `Bash(*)`/`Write(*)` allow-all and Windows-era denies (`format`, `diskpart`) — tidy.

## 5. Unknowns and how to close them

| Unknown | Cheapest resolution |
|---|---|
| Does `@astrojs/sitemap`'s `i18n` option emit alternates when `prefixDefaultLocale: false`? | T3 verify step (`grep xhtml:link dist/sitemap-0.xml`); fallback: hand-emit alternates in BaseLayout only and drop the sitemap option |
| Does Cloudflare Pages honour `_redirects` splats for `/projects/*` on a Direct Upload project? | staging deploy after T3, `curl -sI staging…/projects/memogenesis` → 301 |
| Playwright/Chromium and `lighthouse` present on the VPS? | `npx playwright --version; npx lighthouse --version` — if absent, phone check by hand |
| Generator's Brevo module wording | re-run datenschutz-generator.de with the same modules + Brevo; diff |
| Existing selfie survives the crop? | Luis crops it; if the car shows, About ships without a photo (brief) |

## Next action

On the VPS as `dev`, in the repo: `git switch -c site/t1-tokens`, paste prompt
T1 with `<HEX FROM LUIS>` = `#1F6FEB`.
