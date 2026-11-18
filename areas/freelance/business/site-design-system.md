---
type: standard
status: active
owner: architect
created: 2026-09-27
updated: 2026-11-18
tags: [website, design]
---

# neurogenesis.dev — design system (v3)

> [!NOTE] Summary
> The one set of rules every page follows. v1 came from the Mentor session
> ([[web-presence]] § "Design rules + copy for the site-wide sweep"); v2 (2026-09-27)
> applied Luis's review of the R3 staging build. **v3 (2026-11-18) replaces the five
> coloured bands with a dark system built from three measured references**
> (linear.app token architecture, dennissnellenberg.com page choreography and hairline
> discipline, nateherk.com ambient background) after Luis's decisions of 2026-11-09.
> **The token source is `DESIGN.md` at the repo root** (Google DESIGN.md format, lints
> clean); this note holds the rules, the page inventory and the acceptance gate. Where
> the two disagree, DESIGN.md wins for values and this note for structure. A page that is
> not in the inventory below does not ship. v2 is in the vault's git history.

## 1. Principles

1. **Every page has the same skeleton:** intro → sections → footer, stacked sheets on elevation levels.
2. **A colour means one thing.** Black, five greys, one blue (buttons, the header Contact link, the wordmark). Nothing else is coloured except screenshots and media. The white reading sheet is the single light surface (articles, legal bodies).
3. **Modules, not pages.** Anything that appears on more than one page is one component with one spec. Pages compose modules; they never restyle them.
4. **Copy has one source.** Strings come from `src/i18n/copy.ts`, which mirrors web-presence.md. A page never hardcodes text.
5. **Content may repeat across pages, never within one.**
6. **One motion temperament: ease-out.** What you cause answers on the first frame and settles; what enters arrives and settles. Two moving layers never share a viewport.

## 2. Tokens and levels

Values live in `DESIGN.md` and `src/styles/global.css`. The names below are the Tailwind spellings.

| Band tone (`Band.astro`) | Level | Ground | Used for |
|---|---|---|---|
| `band-intro` | 0 | `#0A0A0A` | first band on every page; on home the full-viewport hero over the ambient loop |
| `band-level-0` | 0 | `#0A0A0A` | a section on the page ground |
| `band-level-1` | 1 | `#101112` | a section that reads as its own sheet |
| `band-booking` | 2 | `#161718` + strong top hairline | **the one action that matters: book a call** — Booking module only: home, /services, /contact, last before the footer |
| `band-reading` | white | `#FFFFFF`, ink `#1C1D20` | article and legal bodies only (luis, 2026-11-09: the single light exception) |
| `band-footer` | 0 | `#0A0A0A` + hairline | every page |

- **Text ladder** on the dark levels: `paper-0` `#F7F8F8` headings and links, `paper-1` `#A3A3A3` body, `paper-2` `#8A8F98` meta and captions, `paper-3` `#62666D` decoration only. On the reading sheet: `ink-0/1/2`.
- **Contextual utilities**: `text-fg-0/1/2`, `border-line` / `line-2` / `line-strong`, `bg-surface-up` / `surface-up-2` read the band they sit in, so one Card, PostCard or TagList works on every level and on the white sheet. A component never names `paper-*` or `ink-*` directly.
- **Hairlines** are the text colour at low alpha (white 8 / 14 / 22 %, ink 17.5 %), never a solid grey. Levels before shadows: a card is one level up with a hairline. Only popovers, the search modal and the mobile menu cast (`shadow-stack`).
- **Accent**: Linear's indigo `#5E6AD2` = `rgb(94, 106, 210)` (luis, 2026-11-09). Every button, the header Contact button-link and the wordmark (text + border) use that exact colour — one token, no hover shade, no second blue. Never a surface, never running text, never a link. White on it 4.7:1; on level 0 it is 4.2:1, fine for the logotype (WCAG exempts logotypes) and the border, which is why it is never text.
- **Type**: Inter Variable with `cv01` + `ss03`, weight stops 510 (labels, buttons) and 590 (every heading; Tailwind's `font-medium` / `font-semibold` are remapped to these). Tracking −0.022em on headings, +0.08em on captions. Display `clamp(2.25rem, 6vw + 1rem, 3.5rem)` for the hero H1 and the marquee. Prose 1.0625rem / 1.7 in the 38 rem column.
- **Motion**: speeds quick .1 s / regular .16 s / slow .25 s / reveal .7 s / entrance 1 s; curves ease-out-quint (movement), ease-out-quad (colour; the default for every `transition-*`), ease-out-expo (entrances), ease-in-quart (the page-transition cover). Tailwind's symmetric default curve is retired. Reduced motion keeps opacity fades, drops every translation, stops the marquee and the loop.

**Adjacency is the primary rule**: two adjacent bands never share a level (the footer sits behind a hairline and is exempt). `check-design.mjs` fails a page that breaks it.

## 3. Type, alignment, width

- Band headings (H1 in an intro, H2 opening a band) are centred on the page. **Exception: the home hero is left-aligned** (Dennis / linear.app/intake model).
- Text blocks are centred on the page as a column (reading column 608px; grids and lists use the container width); text inside is left-justified. Container 64 rem with `clamp(1rem, 4vw, 2.5rem)` side padding.
- Headings inside a text column (/about sections, FAQ questions, article and legal prose) align with the left edge of their column.
- Cards and step boxes: title and text left-aligned inside the box; boxes in one row have equal height; 12 px radius.

## 4. Links and buttons

- Button = accent fill, white label at 510, 6 px radius, 40 px tall (32 px in the header). Booking button label everywhere: EN "Book a call" / DE "Gespräch buchen". Max two buttons per page inside `main`.
- Text links: the surface's text colour + underline on hover/focus only for titles in lists (project titles), always underlined in running text. No blue text links.
- "→" links are text links.
- Hover answers at .16 s: nav links to text-primary, cards to the next level with a stronger hairline, project rows to a translucent ground. No magnetic buttons.

## 5. Modules

**Hero (home only, `Hero.astro`)** — the full viewport (`min-h-svh`, the fixed 4 rem header overlays it) on level 0 over the **ambient loop**: a 15 s video at 60 % opacity, masked back to level 0 before the hero ends, poster first and alone under reduced motion. Text left-aligned in the lower half: eyebrow in caption type ("Luis Gordon · Frankfurt am Main"), **the role line as H1** in display type, one sentence at text-secondary, no button; a small scroll cue bottom right. Entrance: eyebrow, H1, sentence rise 24 px over 1 s on ease-out-expo, 0.08 s apart. **The three target groups leave the hero**; the offer line stays the /services H1. Then the **marquee** (`Marquee.astro`): the name across a full-width strip between hairlines, moved by the scroll itself (CSS view timeline), still at rest, static where unsupported. If the page feels busy on staging the marquee goes, not the loop (luis, 2026-11-09).
Loop production: `design-refs/tools/ambient/` renders the Paper Shaders warp shader (stripes, the four shader greys, quarter speed) frame by frame in sandboxed headless Chromium, cross-fades the tail into the head and encodes AV1 / VP9 / H.264 plus the JPEG poster into `public/media/`. Parameters in `DESIGN.md` § Elevation & Depth.

**Intro (every other page, `Intro.astro`)** — level 0, content height, H1 + one line, centred. /services: offer line as H1, expansion as line, no button. Result pages: H1 only, the message on level 1 below. /about: `AboutIntro.astro` (headshot, name, city) unchanged in layout from v2.

**Header** — 4 rem, fixed, transparent over the hero; once scrolled it fills with level 0 at 80 % behind a 20 px blur plus a hairline (scroll-driven CSS, solid where unsupported). Wordmark left (unchanged box: 2.5–3 px accent border, accent uppercase text, equal padding). Nav in label type at text-secondary, Contact as the accent button, EN/DE switch; mobile menu on level 3 with the stacked shadow.

**Cards** — level 1 section (default); centred H2; cards one level up with a hairline, 12 px radius; hover to the next level; optional "→" link centred under the grid.

**Steps** ("How a project runs") — level 0 section; centred H2; 5 boxes one level up with a hairline, number in caption type, equal height; stack on phones.

**Booking** — level 2 behind a strong hairline; centred: H2 → one line → "Book a call" button. Replaces the old Contact section everywhere. No email in it. Last section before the footer. Copy unchanged from v2 (proposed, Luis to approve): H2 EN "Have a project in mind?" / DE „Sie haben ein Projekt im Kopf?" · line EN "Free, 30 minutes. I usually reply within one working day." / DE „Kostenlos, 30 Minuten. Antwort in der Regel innerhalb von 1 Werktag."

**Newsletter** — centred H2 "Get the next article by email" / „Den nächsten Artikel per E-Mail" → Kit form (level-1 input with a hairline, accent button) → privacy line. Level 1 on /newsletter and under an article (white sheet above); level 0 on /articles once the list is above it. Validity messages in success / danger only.

**Project list** (brittanychiang.com model) — level 1, one row per project: year in caption type, title with ↗, summary, tag pills; the whole row is the link target; hover lifts the row one level, the title underlines, nothing turns blue. Tags: 3–5 per project, one level of abstraction (language / framework / platform / core tool).

**Wordmark** — unchanged from v2 except the colour: "NEUROGENESIS DEV" in a square box, 2.5–3 px border, equal padding, text and border in the accent.

**FAQ list** — level 1, 608px column, hairline-separated, question left.

**Contact form** — level 1; inputs one level up with a hairline, focus moves the border to the text colour and adds a 2 px accent ring; Turnstile invisible. Loads fully (no client-side routing into or out of /contact).

**Footer** — level 0 behind a hairline, full container width. Left: page list in label type (Services · Projects · Articles · About) before the vertical divider. Right: three columns with caption headings: **Updates** (Newsletter · Articles) · **Contact** (Contact · FAQ) · **Legal** (Impressum · Privacy/Datenschutz). Full-width hairline, © left, five social icons right. < md: stacked.

**Page transition** — Astro `ClientRouter`: a level-0 panel covers from the bottom over .4 s on ease-in-quart, the new page rises and fades in over .5 s on ease-out-quint (under a second). Built as drafted, **to be judged on staging** (luis, 2026-11-09). Every section's content rises 2 em over .7 s when it enters the viewport, once.

## 6. Page inventory (EN and DE twin for every row unless marked)

| Route | Bands in order |
|---|---|
| `/` `/de/` | hero (full viewport, loop, role line, marquee) → cards "What I offer" + "All services →" (1) → "Automation in action" (0) → "Projects" teaser + "All projects →" (1) → **booking** (2) → footer |
| `/services` | intro (offer line, expansion, no button) → cards + price line (1) → steps (0) → **booking** (2) → footer |
| `/projects` | intro → project list (1) → footer |
| `/articles` | intro → [article list (1), once posts exist] → newsletter (0, or 1 while the list is absent) → footer |
| `/newsletter` | intro → newsletter (1) → footer |
| `/about` | intro (headshot, name, city) → "What I do", "The name", "Interests" (1) → "Socials" (0) → footer. **No booking, no contact.** |
| `/contact` | intro (H1 + line) → email line + form (1) → **booking** (2) → footer |
| `/contact/sent`, `/contact/error` | intro (H1 only) → message (1) → footer |
| `/faq` | intro → FAQ list (1) → footer |
| `/impressum`, `/privacy` / `/de/datenschutz` | intro (Stand line) → body (reading, white) → footer |
| article pages (EN) | intro → body (reading, white) → newsletter (1) → footer |
| 404 | intro → footer |

**Spelling (R5):** American English in all EN copy.

**Hero copy (proposed 2026-11-18, Luis to approve):** H1 EN "Freelance web developer & automation engineer" / DE „Freiberuflicher Web- & Automatisierungsentwickler"; line EN "Websites and workflows for small businesses and independents – built, explained and kept running by one person." / DE „Websites und Workflows für kleine Unternehmen und Selbstständige – gebaut, erklärt und am Laufen gehalten von einer Person."; cue "Scroll" / „Scrollen". Lands in web-presence.md once decided.

## 7. Contact form (/contact)

Unchanged from v2 in behaviour (Pages Function → Brevo, Turnstile invisible, sender `contact@neurogenesis.dev` "neurogenesis.dev contact-form", privacy sections in both languages). Band: level 1. The email line sits above the form in the form's column (R5). Contact links carry `data-astro-reload` and the form posts with a full load so the Turnstile script is never re-run by the client router.

## 8. Acceptance

1. Screenshot every built route at 390 and 1280 (full page), Read each; no overflow at 320.
2. `check-design.mjs`: intro first, footer last, no adjacent same tone or same level; booking band only on /, /de/, /services, /de/services, /contact, /de/contact, once each; ≤ 2 buttons; retired strings and tokens absent (`band-content`, `surface-0`, `#1F6FEB`, `#16263B`, `#EEEAE6`, …); the home pages open with the full-viewport hero, ambient layer and marquee; **wordmark border, wordmark text, header Contact and every button resolve to `rgb(94, 106, 210)`** in the browser.
3. `check-contrast.mjs`: the text ladder on every level, ink on the reading sheet, white on the accent, the accent on every surface it sits on.
4. `DESIGN.md` lints clean (`npx @google/design.md lint DESIGN.md`).
5. Staging; Luis judges the loop, the marquee and the page transition in the browser; newsletter + contact form tested live.
