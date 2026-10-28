---
type: standard
status: active
owner: architect
created: 2026-09-27
updated: 2026-09-27
tags: [website, design]
---

# neurogenesis.dev — design system (v2)

> [!NOTE] Summary
> The one set of rules every page follows. v1 came from the Mentor session
> ([[web-presence]] § "Design rules + copy for the site-wide sweep"); **v2
> (2026-09-27 evening) applies Luis's review of the R3 staging build** and
> overrides v1 where they differ: navy returns as the booking band, the
> contact section is replaced by it, the footer gets a Legal column, the home
> hero shrinks. A page that is not in the inventory below does not ship.

## 1. Principles

1. **Every page has the same skeleton:** intro band → content band(s) → footer.
2. **A colour means one thing.** Five band colours, one accent. Nothing else is coloured except screenshots and media.
3. **Modules, not pages.** Anything that appears on more than one page is one component with one spec. Pages compose modules; they never restyle them.
4. **Copy has one source.** Strings come from `src/i18n/copy.ts`, which mirrors web-presence.md. A page never hardcodes text.
5. **Content may repeat across pages, never within one.**

## 2. Tokens and bands

| Band | Colour | Meaning | Used for |
|---|---|---|---|
| `band-intro` | gradient `#0a0a0a` → `#4a4a4a` | top of a page | first band on every page |
| `band-content` | `#EEEAE6` off-white | browse | cards, lists, forms, the article list |
| `band-reading` | `#FFFFFF` white | read | article and legal bodies, process steps, newsletter, /about socials, home demo |
| `band-booking` | `#16263B` navy | **the one action that matters: book a call** (luis, 2026-09-27 — like Chris Williamson's newsletter band) | Booking module only: home, /services, /contact |
| `band-footer` | `#2B2B2B` dark grey | end | every page |

Accent: **the colour the header Contact button renders today is the reference** (luis). Every button, the header Contact button-link and the wordmark (text + border) use that exact rendered colour — one token, no opacity, no second blue. Never a band, never text links.

**Band order is the primary rule** (luis, 2026-09-27): intro gradient → off-white → white → (alternate further if needed) → footer, with navy placed where the Booking module sits; no two adjacent bands of the same colour; aim to use every band colour on a page where content allows. The meanings above (browse / read) are secondary — they decide ties, they never force a band out of order.

## 3. Type, alignment, width

- Band headings (H1 in an intro, H2 opening a band) are centred on the page.
- Text blocks are centred on the page as a column (reading column 608px; grids and lists use the container width); text inside is left-justified.
- Headings inside a text column (/about intro and sections, FAQ questions, article and legal prose) align with the left edge of their column.
- Cards and step boxes: title and text left-aligned inside the box; boxes in one row have equal height.

## 4. Links and buttons

- Button = accent fill, white label. Booking button label everywhere: EN "Book a call" / DE „Gespräch buchen". Max two buttons per page.
- Text links: band text colour + underline on hover/focus only for titles in lists (project titles), always underlined in running text. No blue text links.
- "→" links are text links.

## 5. Modules

**Intro** — `band-intro`, content height on every page except home. H1 + one line. **Home (R5, 2026-09-28):** hero ~60–70 % of the viewport height, H1 "Luis Gordon" stays the headline, offer line one type step larger than today, text centred vertically, **no button** (the booking band follows directly). /services: offer line as H1, expansion as line, **no button**. **/contact (luis, 2026-09-28, Chris Williamson model):** the intro band keeps H1 + intro line like every page. The email line (R5 copy) is a left-justified paragraph in the off-white band, in the form's column directly above the form. **Result pages** (/contact/sent, /contact/error + DE): intro with H1 only, the message on an off-white band below.

**Gradient (R5, luis):** the black→grey transition starts at ~50–60 % of the band height and ends lighter than `#4a4a4a` (more grey toward white); every text on it keeps ≥ 4.5:1.

**/about (R5, 2026-09-28)** — all headings on /about left-aligned with their text, Socials included. Two variants for Luis to pick from screenshots: **A** intro band = headshot + name + city only; What I do, The name, Interests on off-white; Socials on white. **B** everything except Socials in the gradient band (headshot left, text column as in v2); Socials on white. **Chosen (luis, 2026-09-28): A with the "center" intro** — name + city block centred on the page axis like every other intro, 240px headshot to its left at 32px (profile photo left of the name, per social-media convention); the text blocks below keep their own left edge. Phones: 88px photo beside name + city. Copy: web-presence.md § R5 (replaces the 2026-09-27 /about copy; no "Currently" line; Threads removed).

**Cards** — `band-content`; centred H2; 3 white cards; optional "→" link centred under the grid.

**Steps** ("How a project runs") — `band-reading`; centred H2; 5 **off-white boxes**, one per step, same box style as the cards (inverse colour), number + text, equal height; stack on phones.

**Booking** — `band-booking` (navy); centred: H2 → one line → "Book a call" button. Replaces the old Contact section everywhere. No email in it.
Copy — **proposed, Luis to approve:** H2 EN "Have a project in mind?" / DE „Sie haben ein Projekt im Kopf?" · line EN "Free, 30 minutes. I usually reply within one working day." / DE „Kostenlos, 30 Minuten. Antwort in der Regel innerhalb von 1 Werktag." (one line, identical everywhere).

**Newsletter** — centred H2 "Get the next article by email" / „Den nächsten Artikel per E-Mail" → Kit form → privacy line. `band-reading` on /articles and /newsletter; on article pages `band-content` (the body above is white — sequence rule).

**Project list** (brittanychiang.com model, `references/chiang-work-1280.png`) — `band-content`, one row per project: left column date range (small caps, muted); right: title with ↗ (no underline at rest, underline + accent-free highlight on hover), summary, then tag pills. Whole row is the link target; hover lifts the row with a subtle white background. Tags: **3–5 per project, same level of abstraction** (language / framework / platform / core tool — never a CI detail).
Tags per project (luis approved 2026-09-27): **neurogenesis.dev** entry: Astro · TypeScript · Tailwind CSS · Cloudflare Pages. **Automation stack** entry: n8n · Claude API · Podman · Debian. Allowed tag levels for future entries: language, framework, platform/host, core tool — never CI, libraries or config details.

**Wordmark** — "NEUROGENESIS DEV" in a square box: border 2.5–3px (one step thicker than today), padding equal on all four sides (horizontal = today's vertical), text and border in the accent reference colour.

**FAQ list** — `band-content`, 608px column, hairline-separated, question left.

**Footer** — `band-footer`, full container width. Left: page list in bold (Services · Projects · Articles · About), generous space before the vertical divider. Right: **three** columns with small bold headings, in this order: **Updates** (Newsletter · Articles) · **Contact** (Contact · FAQ) · **Legal** (Impressum · Privacy/Datenschutz). No "Book a call" or email in the footer (never confirmed by Luis). Full-width divider, then © left, six social icons right. < md: stacked.

## 6. Page inventory (EN and DE twin for every row unless marked)

| Route | Bands in order |
|---|---|
| `/` `/de/` | intro (compact: name at normal H1 size, offer line) → **booking** → cards "What I offer" + "All services →" (content) → "Automation in action" (reading) → "Projects" teaser + "All projects →" (content) → footer |
| `/services` | intro (no button) → cards + price line (content) → steps (reading) → **booking** → footer |
| `/projects` | intro → project list (content) → footer |
| `/articles` | intro → [article list (content), once posts exist] → newsletter (reading) → footer |
| `/newsletter` | intro → newsletter (reading) → footer |
| `/about` | variant A or B (§5) → "Socials" left-aligned list (reading) → footer. **No booking, no contact.** |
| `/contact` | intro (H1 + line) → email line + form (content) → **booking** → footer |
| `/faq` | intro → FAQ list (content) → footer |
| `/impressum`, `/privacy` / `/de/datenschutz` | intro (Stand line) → body (reading) → footer |
| article pages (EN) | intro → body (reading) → newsletter (content) → footer |
| 404 | intro → footer |

**Spelling (R5):** American English in all EN copy (optimization, color, center, behavior, license).

## 7. Contact form (/contact)

Unchanged from v1 (Pages Function → Brevo, Turnstile, sender `contact@neurogenesis.dev` "neurogenesis.dev contact-form", privacy sections in both languages). Band: `band-content`. The email link moves into the intro (R5). **Turnstile runs in Invisible mode** (luis, 2026-09-28) → no widget space in the form, and both privacy texts must reference Cloudflare's Turnstile Privacy Addendum (https://www.cloudflare.com/turnstile-privacy-policy/).

## 8. Acceptance

1. Screenshot every built route at 390 and 1280 (full page), Read each; no overflow at 320.
2. `check-design.mjs`: intro first, footer last, no adjacent same band; booking band only on /, /de/, /services, /de/services, /contact, /de/contact; ≤ 2 buttons; forbidden strings (`Folgt in Kürze`, `Stay in the loop`, "30-minute" inside a link/button); **wordmark border, wordmark text, header Contact and every button resolve to the same computed colour** (check in the browser, not only CSS source).
3. `npm run lint` incl. contrast pairs (white on navy, muted on navy).
4. Codex; staging; Luis review; newsletter + contact form tested live.
