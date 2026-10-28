---
type: decision
id: ADR-0038
aliases: [ADR-0038]
status: accepted  # 2026-09-21, Luis confirmed the four flags (flag 4 amended: post URLs move to /articles/)
provenance: inferred
device: shared
created: 2026-09-21
updated: 2026-09-21
tags: [website, astro, i18n, seo, legal, fonts, neurogenesis.dev, freelance]
---

# ADR-0038 — neurogenesis.dev: bilingual routing (TDR addendum to FDR-007)

> [!NOTE] Summary
> Technical side of `freelance-practice` FDR-007 (bilingual EN/DE, EN at `/`,
> DE under `/de/`, no auto-redirect, articles EN-only). Astro's built-in i18n
> routing, no library; per-page hreflang + x-default; a rule-based language
> switch; posts move to `/articles/<slug>` with 301s from `/blog/*`; German legal texts
> binding, English ones marked convenience translation; Brevo added to both
> privacy pages; Inter + JetBrains Mono under the SIL OFL, licence files
> shipped. Implementation tasks: `freelance-practice/business/site-redesign-round-one.md`.

## Context

Site: Astro 6.4 static, Tailwind v4, Cloudflare Pages ([[../runbooks/neurogenesis-dev-site]]).
Today: EN blog/home/about at `/`, DE `/impressum`, `/datenschutz`, `/contact`,
`/services` (noindex) at `/`, EN `/privacy` as courtesy. FDR-007 flips the
split to bilingual with a switch. The redesign brief caps round one at 15 h,
so the routing must be cheap to build and impossible to half-finish: every
task leaves a deployable site.

## Decision

**Routing — Astro built-in i18n, no plugin.**

```js
// astro.config.mjs
i18n: {
  defaultLocale: 'en',
  locales: ['en', 'de'],
  routing: { prefixDefaultLocale: false },   // EN at /, DE at /de/
  // no `fallback`: a missing DE page 404s instead of silently serving EN
},
integrations: [ sitemap({
  i18n: { defaultLocale: 'en', locales: { en: 'en', de: 'de' } },   // emits xhtml:link alternates
  filter: (p) => !/\/(de\/)?services\/?$/.test(p),                  // FDR-005 gate, both languages
}) ]
```

- Files: `src/pages/<page>.astro` (EN) and `src/pages/de/<page>.astro` (DE),
  each a thin wrapper that renders one shared page component with `lang`
  and the copy object. UI strings (nav, footer, buttons, switch) in
  `src/i18n/ui.ts`; page copy lives in the page files — 6 short pages, a
  content collection would be ceremony.
- No auto-redirect by `Accept-Language`: static host, the URL decides
  (FDR-007). Cloudflare Pages does no language redirects by default — nothing to
  disable.
- `<html lang>` from `Astro.currentLocale` in `BaseLayout`.

**hreflang.** `BaseLayout` takes `alternates?: { en: string; de: string }`.
When present it emits `rel="alternate" hreflang="en"`, `hreflang="de"`, and
`hreflang="x-default"` → the EN URL (absolute, from `site`). Pages without a
counterpart (articles, tags, changelog, newsletter) emit none. Sitemap
alternates come from the integration's `i18n` option; both must agree — QA
task greps the built HTML against `sitemap-0.xml`.

**Language switch — `LangSwitch.astro`, no JS, no cookie.** Counterpart URL
by rule from `Astro.url.pathname`: `/x` ↔ `/de/x`; a page can override with a
`switchTo` prop. On EN-only pages (`/articles`, `/blog/*`, `/tags/*`,
`/changelog`, `/newsletter`) the DE link points to `/de/` and carries
`title="Artikel gibt es nur auf Englisch"`. Switch stays visible everywhere
(brief: visible switch), so a German visitor is never stranded.

**URL map (redirects in `public/_redirects`, 301).**

| Old | New | Why |
|---|---|---|
| `/datenschutz` | `/de/datenschutz` | DE moves under `/de/`; Google OAuth consent screen still points at the old URL — a 301 keeps it valid; update the consent screen when convenient |
| `/impressum` | stays as the EN version; new `/de/impressum` | Impressum content identical, labels translated |
| `/privacy` | stays | EN privacy, convenience translation |
| `/contact`, `/services` | stay (become EN); new `/de/contact`, `/de/services` | existing DE copy moves to `/de/` |
| `/blog` | `/articles` | rename |
| `/blog/<slug>` | `/articles/<slug>` (`/blog/* /articles/:splat 301`) | Luis 2026-09-21: Giscus threads and RSS GUIDs have no readers, so the orphaning cost is nil; one URL scheme is cleaner. RSS feed path stays `/rss.xml` |
| `/projects`, `/projects/*` | `/work` | new project list; memogenesis left out (brief) |

**Legal.** German texts are binding. `/privacy` and `/impressum` (EN) open with
one line: "Convenience translation — the German version at /de/… is legally
binding." Both privacy pages get the **Brevo** module (Brevo / Sendinblue GmbH,
Köln, EU; purpose: outbound business email relay for `@neurogenesis.dev`;
legal basis Art. 6(1)(b)/(f) GDPR; data: sender/recipient address, subject,
content in transit). Source: re-run datenschutz-generator.de with the Brevo
module ticked and diff against `src/content/legal/datenschutz.md`, rather than
hand-writing; the free licence still applies (revenue < €5k/yr). EN text:
translation of the generator output, same `updated` date as the DE file.
Rule from the runbook stands: new processor → same commit updates both files.

**Fonts.** Inter — SIL Open Font License 1.1 (rsms/inter; already self-hosted
as `inter-variable.woff2`). Mono: keep **JetBrains Mono** — SIL OFL 1.1,
already in `public/fonts/`, zero work; Geist Mono is also OFL 1.1 but adds a
file for no gain. OFL obligations: ship the licence texts
(`public/fonts/OFL-Inter.txt`, `public/fonts/OFL-JetBrainsMono.txt`), do not
sell the fonts alone, keep Reserved Font Names if the files are modified
(they are not). No serif. Copy the licence files from the upstream repos
(Inter: github.com/rsms/inter `LICENSE.txt`; JetBrains Mono:
github.com/JetBrains/JetBrainsMono `OFL.txt`).

**Four flags — confirmed by Luis 2026-09-21:**

1. Remove the light/dark **theme toggle** — yes.
2. Remove **ClientRouter** (view transitions: pages fade instead of reloading, at
   the price of re-initialising every script after each fade — a known bug
   source here) — yes; plain page loads, CSS transitions only. Evidence (luis):
   the obfuscated email on /impressum only appears after a refresh — the
   assembling script never re-runs after a client-side navigation.
3. Article/reading pages on the **light** band (white background, dark text,
   dark code blocks) — yes.
4. Post URLs → `/articles/<slug>` with 301s (amended from "stay"; table above).

Accent blue chosen: **`#1F6FEB`** (option B).

## Rejected

- **`[lang]` dynamic route with `getStaticPaths`** — puts EN under `/en/` or
  needs rewrite tricks to keep EN at `/`; every page becomes a template with
  a locale loop. More clever, harder to read at 2 a.m.
- **i18n library (astro-i18next, Paraglide, astro-i18n)** — dependency and
  build step for six pages of copy. Astro's built-in routing covers it.
- **Browser-language redirect** (Pages Function or `<meta refresh>`) —
  FDR-007 says the URL decides; also adds a Function to a static site.
- **Keeping post URLs at `/blog/<slug>`** — Architect's first proposal (protects
  Giscus threads and RSS GUIDs); dropped because neither has readers and two
  URL schemes for one section is confusing.
- **DE `fallback: { de: 'en' }`** — would mask a missing German page as
  English content under a `/de/` URL; wrong signal to German prospects and
  to search engines.
- **Geist Mono** — fine licence, no advantage over the mono already shipped.

## Consequences

- Every business-page copy change is made twice (FDR-007 accepted this).
- `src/pages/de/` doubles the page count; the shared-component pattern keeps
  logic in one place. Lint: a script asserts every `src/pages/de/X` has an
  `src/pages/X` twin and vice versa (excluding EN-only routes).
- OG images (satori) take the page title per locale — no change needed.
- Cloudflare Web Analytics, Kit form, Cal.com link unchanged; the Kit form's
  privacy link becomes `/privacy` on EN pages and `/de/datenschutz` on DE pages.

## Related

- `freelance-practice/decisions/fdr-007-bilingual-site-with-language-switch.md`
- `freelance-practice/business/site-design-brief.md`, `site-redesign-round-one.md`
- [[../runbooks/neurogenesis-dev-site]] — build gate, deploy, legal pages, backlog
