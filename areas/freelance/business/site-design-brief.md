---
type: note
status: active
owner: mentor
created: 2026-09-21
updated: 2026-09-21
tags: [business, marketing]
---

# Site design brief — neurogenesis.dev redesign

> [!NOTE] Summary
> Discovery output for the redesign of neurogenesis.dev (Mentor ↔ Luis,
> 2026-09-20/21). Round one: Linear-style dark first screen, light sections
> with dark media, greyscale + one blue accent on the booking action, Inter,
> bilingual EN/DE, conventional structure with minimal motion. Hard cap 15 h.
> Everything Luis wants beyond round one is in **Deferred to later rounds**.
> Handed back to the Architect 2026-09-21 for design inventory, TDR addendum
> (i18n) and the Claude Code task list. Copy rules, legal facts and the
> /services gate live in [[web-presence]],
> [[../decisions/fdr-004-business-identity-on-neurogenesis-dev|FDR-004]],
> [[../decisions/fdr-005-no-public-prices-until-first-paid-quote|FDR-005]] and
> [[../decisions/fdr-007-bilingual-site-with-language-switch|FDR-007]].

## Purpose & ranking (luis, 2026-09-20)

The site promotes the business first; it is not primarily a personal site.
Presenting Luis and his work well serves that.

1. Give a prospect enough confidence to reply or book the 30-minute call
2. Portfolio (own projects now; Chantal's case study after launch, consent pending)
3. Articles as supporting proof of taste and seriousness — "the ornament in the room"
4. Legal layer: must exist, footer only, no influence on design

Hard requirement: within 5 seconds a cold visitor understands **what is offered
and for whom**.

Until testimonials exist, confidence comes from: the site itself being visibly
well made, a clear offer and process, name + honest "About", 1–2 real projects
with screenshots, automation shown in action.

## Audiences

| Visitor | Arrives via | Lands on | Needs |
|---|---|---|---|
| Contacted prospect | outreach message from Luis | /de/services or /services (language of the outreach) | offer, process, booking |
| Referred prospect | word of mouth | home or /services | "is he legit", booking |
| Social-profile clicker (friends, family, followers, new contacts) | link on a neurogenesisdev profile | home | what he does, in 5 s |
| Tech reader | articles, Reddit, GitHub | /articles | the writing; business is a side note |

Language ([[../decisions/fdr-007-bilingual-site-with-language-switch|FDR-007]]):
bilingual EN/DE with a visible switch for home, /about, /services, /contact
and legal pages; articles English-only. English at `/`, German under `/de/`,
no auto-redirect. /services works as a **standalone landing page** in both
languages (own mini-hero, short About block, booking) because outreach links
there directly. One visual identity for business pages and articles.

## Visual direction

### References (luis, 2026-09-21)

| Site | Verdict | Take / avoid |
|---|---|---|
| linear.app | 👍👍 main reference (already inspired the current site, with Vercel) | black→grey gradient first screen; very dark yet professional; static + moving UI screenshots give the tech-savvy feel |
| dennissnellenberg.com | 👍 first screen | 3/4 portrait looking aside (pose he'd accept in a shoot); sliding full-name banner; right-aligned nav with 3 items; "Located in …" tag; a single blue circle behind the contact link as the only colour. Avoid: cursor-following previews, forced scroll pop-ins |
| Tim Ferriss, Chris Williamson | 👍 structure | full name standalone at the top; short line under it; sticky nav that does not blur content; full-width sections with distinct backgrounds (dark top → lighter → grey footer). Avoid (for now): full-screen face portrait |
| hubermanlab.com | 👍 parts | white that doesn't glare because large dark media tiles fill it; scrolling praise window (later testimonial model) |
| brittanychiang.com | 👍 parts | subdued cursor glow = acceptable level of effect; project timeline with tech tags → model for /work. Avoid: reads as a résumé |
| joshwcomeau.com | ± | smooth animation; interactive piece confined to the hero is the right dose. Avoid: quirky/experimental tone |
| stephango.com | ± | short posts = reachable model for his own articles. Avoid: too plain, "hermit" tone |
| gwern.net | ± (content 👍) | category + reverse-chronological article lists, later. Avoid: tome-like density |
| maggieappleton.com | 👎 | no brown/warm solid background, no giant serif intro, no sections made by font size alone |
| rauno.me | 👎👎 | no journey/storytelling sites |
| lexfridman.com | 👎 | bare white page, text only, cluttered with hyperlinks |

Principle (luis): look like sites the general public already reads as
professional; "a toe outside the box, not the whole body".

### Constraints (round one)

- **Colour:** black, white, shades of grey. Exactly one accent, **blue, never purple**, reserved for the booking/contact action (button or a Snellenberg-style circle behind the link). Exact blue: Architect proposes 2–3, Luis picks. Screenshots are the only other colour on the page
- **Type:** Inter (OFL, self-hosted) for everything; an open-licence mono for technical text (e.g. JetBrains Mono or Geist Mono — Linear's Berkeley Mono is a paid licence, not used). Architect verifies licences. No serif
- **First screen:** full-height black→grey gradient (Linear-style) with "Luis Gordon", the offer line, booking button. No portrait
- **Name:** top-left "Luis Gordon" replaces "neurogenesis.dev" (domain/handle remain the brand in URL and socials)
- **Line under the name states the offer**, not a personal slogan
- **Nav:** sticky, solid background, no blur, right-aligned, few items — Services · Work · Articles · About + Contact (accent) + EN/DE switch
- **Sections:** full-width bands; dark opening and nav → light content sections → grey/dark footer. Light sections carry large **dark media** (dark-mode screenshots: demo Gmail inbox being sorted, n8n workflow canvas, Obsidian graph; later client sites) so white never glares
- **Motion:** smooth, subdued transitions. One cursor-reactive effect on the first screen is a *could*, built last, with static fallback on touch devices and `prefers-reduced-motion`. No scroll-jacking, no cursor-following previews, no intro animation
- **/work:** timeline/list of projects with tech tags per entry (Chiang model)
- **Search:** not on home; only on /articles. "Blog" renamed "Articles"
- **About block:** headshot optional in round one (existing dress-shirt selfie only if the crop hides the car; else none until the shoot)
- **Testimonial slot** reserved in the layout; must not read as empty today
- **Automation is shown, not described** — see demo loop below

### Demo loop (luis, 2026-09-21)

The n8n Gmail classifier runs today. Record it on a **dedicated demo Gmail
account with a fabricated inbox**, not a sanitised view of the real one (blur
looks cheap and leaks eventually; the demo account is reusable for client
demos). Fabricated mails must not imply clients or real companies. Output:
one short silent loop (inbox sorting itself) + one static n8n canvas
screenshot, dark mode. Telegram "secretary" bot: backlog until it exists.

## Content inventory (honest, as of 2026-09-21)

- Two own websites (this one included)
- Automation stack: n8n Gmail classifier (demo loop); others per infra-docs
- Chantal's site: after launch and written consent only
- **Memogenesis: left out of round one** (luis, 2026-09-21). Returns when it runs again
- Existing articles were written by Claude, not by Luis → not featured as proof; /articles is a nav item only, restyled, not reorganised
- Wording rules: [[web-presence]] — apply in both languages (no claimed experience, no plural "Kunden"/"clients", no "Agentur"/"agency")

## Non-goals (round one)

- Fixed by tech: Astro on Cloudflare Pages, no tracking / consent banner, legal pages stay, fonts self-hosted, /services gate unchanged (both languages), no CMS, no framework change
- No logo, no new photo, no testimonial content, no writing strip, no article reorganisation, no Memogenesis, no contact form

## Time box & done criteria (luis + mentor, 2026-09-21)

**Hard cap 15 h** (Architect's 12 h + bilingual), logged in `hours.csv` as
`site`. Luis's stance: he doesn't mind the time — his part is taste decisions
and orchestrating Claude Code. Mentor's stance: orchestration hours are hours;
they count toward the 20 h/week flag and the employer's Ruhezeiten condition,
and Chantal's site has priority. At 15 h: ship what exists, move the rest to
the backlog.

Done for round one:

- [ ] Home, /services, /work, /about, /contact in the new design, EN and DE, language switch works
- [ ] Legal pages and /articles restyled (content unchanged; EN legal = convenience translation)
- [ ] Demo loop + n8n screenshot on home and /services
- [ ] Passes on a phone: offer readable and booking reachable on the first screen
- [ ] Wording rules hold in both languages; /services still unlinked + noindex until the gate opens
- [ ] Could, only with hours left: cursor effect on the first screen

## Deferred to later rounds (backlog — everything Luis wants that round one excludes)

| Item | Note | Trigger |
|---|---|---|
| Logo | N partially enclosed by a half circle opening to the left — reads as a D without its vertical stroke, and as a vesicle releasing neurotransmitters. Text wordmark until then | after round one ships |
| Professional headshot | Luis dislikes being photographed; treat as a comfort-zone experiment. Pose brief: Snellenberg-style — 3/4 torso, looking to the side, relaxed, not frontal face | before first paid outreach wave |
| Full-screen portrait first screen | replaces the gradient; with sliding full-name banner and "Located in Frankfurt" tag (spinning globe optional) | with professional headshot |
| Multilingual "hello" intro (Snellenberg) | liked, fits the bilingual site, but delays the message | only if skippable and < 1 s |
| Interactive hero piece with controls (Comeau-style, lines following the cursor, sliders) | skill display; beyond the one subtle effect allowed in round one | after round one, if hours allow |
| Colour-coded section bands (Williamson: dark → blue → off-white → white → grey) | round one stays greyscale; revisit once the accent blue is proven | round two |
| Personal tagline ("Learning out loud"-style) | for /about or /articles, never instead of the offer line | with first self-written article |
| Testimonial strip (Huberman-style scrolling window) | slot reserved in layout | ≥ 3 client quotes |
| Client work as dark media tiles | sites and automation setups, static and moving | per case study |
| Telegram "secretary" bot demo | second automation loop | when the bot exists |
| Writing strip on home (2–3 hand-picked articles) | only self-written pieces; raw material: his long Reddit comments; Steph Ango-length posts as the reachable model | ≥ 2 self-written articles |
| Articles direction | AI/automation/tech/design plus philosophy, cognitive biases, logical fallacies in modern society (LessWrong-style); stays on /articles, never leads the home page | when blogging resumes (1/month) |
| Articles index by category + reverse-chronological (gwern model) | needs content first | ≥ 8 articles |
| Memogenesis on /work | as a running product, not as "inactive"; project itself lives in flashcard-tools-vault | when it runs again |
| Contact form | deferred in web-presence (extra infrastructure); Luis wants a real one (2026-09-25) | first complaint about mailto/Cal.com, or when hours allow |
| Client reference form on neurogenesis.dev | own-domain replacement for the Google Form used for client reference discovery (see clients/chantal-fuelber/reference-addendum); needs answer storage → Architect | after round one ships |
| Contract / terms template for first paid project | not a site item; tracked here so it isn't lost | before first paid quote |

## Open questions (for the Architect / next session)

- [x] Accent blue: `#1F6FEB` (luis, 2026-09-21)
- [x] Mono font: keep JetBrains Mono (OFL 1.1, already shipped); Inter OFL 1.1; licence files added in T1 (Architect, 2026-09-21)
- [ ] Offer line wording EN + DE (Mentor drafts from the /services positioning in web-presence)
- [ ] EN copy for /services and /contact (Mentor drafts; same wording rules)
- [ ] Does the current selfie survive the crop? If not, About block ships without a photo
- [ ] Brevo listed as processor in Datenschutz/Privacy — task T7 in [[site-redesign-round-one]]; wording from a generator re-run (ADR-0038)
- [x] Architect's four flags confirmed (theme toggle out, ClientRouter out, light reading pages, posts move to `/articles/<slug>`) — ADR-0038 accepted (luis, 2026-09-21)
- [x] Hours: count from 2026-09-21, 15–20 h; Luis: Gewerbe not yet approved, business still being built, spends what he wants as long as Chantal's site isn't delayed (luis, 2026-09-21). **Mentor to note** — the 15 h line now means "declare round one shipped", not "stop"
- [x] Fictional enquiry in the demo inbox: include — it shows what a client would get (luis, 2026-09-21)
