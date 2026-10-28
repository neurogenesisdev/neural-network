---
type: note
status: active
owner: mentor
created: 2026-09-14
updated: 2026-09-27
tags: [business, marketing]
---

# Web presence — neurogenesis.dev business section

> [!NOTE] Summary
> Business-side facts and copy decisions for the business pages on
> neurogenesis.dev (/services, /contact, /impressum, /datenschutz), handed to
> the Architect on 2026-09-14. Technical side (Astro, Cloudflare Pages, CSP,
> deploy) lives in `runbooks/` and infra-docs. Legal text comes from a generator,
> not AI prose.

## Impressum (luis, 2026-09-14)

- Name: Luis Gordon (Einzelunternehmer — own name; optional Geschäftsbezeichnung "neurogenesis.dev")
- Address: Am Poloplatz 8, 60528 Frankfurt am Main, Deutschland (home; ladungsfähige Anschrift required, Postfach not allowed; Impressum service ~10–30 €/month rejected at zero revenue — revisit at first paid client)
- Email: luiswg@neurogenesis.dev ([[../decisions/fdr-004-business-identity-on-neurogenesis-dev|FDR-004]])
- Phone: none. Email + contact page satisfy the second-contact-channel requirement
- VAT: no USt-IdNr; no "§19 UStG" line in the Impressum (belongs on invoices only)
- Register/authority: none (no Handelsregister, Aufsichtsbehörde, or Kammer — same for Gewerbe or Freiberufler)
- Responsible for content per § 18 Abs. 2 MStV: Luis Gordon, same address
- Generator must output § 5 DDG wording (not TMG)

## Publication gate (mentor, 2026-09-14)

- Impressum, Datenschutzerklärung, /contact: publish immediately — overdue for the blog (newsletter, comments, analytics = geschäftsmäßig)
- /services: build now, keep unlinked + noindex until (1) employer consent in Textform ([[nebentaetigkeit-anzeige]]) and (2) Gewerbeanmeldung / Fragebogen filed ([[gruendung]]). Advertising paid services counts as starting the business. Target: live within 2–3 weeks of 2026-09-14
- **Gate opened 2026-09-27 (luis):** consent received 2026-09-16, Gewerbeanmeldung filed 2026-09-17 (Tätigkeitsbeginn 17.09.2026), Gewerbeschein pending. /services public and linked in the header (round two). Enquiries before the Gewerbeschein arrives = preparation and contact; no paid work until it is in hand. Fragebogen zur steuerlichen Erfassung due by 2026-10-17 ([[gruendung]] step 4).

## /services copy (German, Sie)

- Positioning: "Websites und Automatisierungen für Selbstständige, Creator und kleine Unternehmen – klar gebaut, in Ihrem Namen, ohne Agentur-Overhead."
- Offers listed: Website (Portfolio/Visitenkarte, bis 5 Seiten, inkl. Impressum & Datenschutz) · Automatisierung (ein Workflow) · Website-Pflege (monatlich). Maps to the catalog in [[../services/_index|services]]
- Prices: none public. "Auf Anfrage — Festpreis nach einem kostenlosen 30-Minuten-Gespräch." Revisit after the first paid quote ([[../decisions/fdr-005-no-public-prices-until-first-paid-quote|FDR-005]])
- Experience statement (verbatim, no "erfahrungsgemäß", no "Kunden" plural, no "Agentur"):
  "Ich baue seit 2025 Websites und Automatisierungen – bisher für eigene Projekte, aktuell die erste Website für eine Kundin. Sie bekommen dafür volle Aufmerksamkeit, direkten Kontakt und einen fairen Einstiegspreis."
- /about (EN) appendix: "Since 2025 I build websites and automations, so far for my own projects — this site and a SaaS in development. My first client project is in progress. I work with a lot of AI tooling, which is how one person delivers what used to take a small team."
- Short "Über mich" block on /services instead of a German /about

**Superseded 2026-09-24 by the T5 section below** (positioning sentence shortened, cards rewritten, process now five steps). The experience statement is kept with two edits ("einen Kunden"; EN "In return").

## /services copy — T5 (luis + mentor, 2026-09-24, settled)

Designed question by question with Luis. Architect builds this unchanged. Rules hold: no claimed experience, no "Kunden"/"clients" plural, no "Agentur"/"agency", Sie, English leads.

### Layout decisions (luis)

- **Header (sticky, all pages):** wordmark `NEUROGENESIS DEV` in a blue outlined box (Huberman-style), top-left, replaces "Luis Gordon" in the header; serves as the interim logo until the real one exists.
- **Home hero:** full name + offer line + booking button (Williamson-style), unchanged from the brief.
- **/services top (dark):** the offer line as H1 (no name) + the expansion paragraph + button. Outreach lands here without seeing the home hero.
- /services stays unlinked + noindex until the gate opens; then "Services / Leistungen" joins the header nav.

### Strings

```
OFFER LINE (home hero + /services H1)
  EN  Websites and automations for independent professionals, small businesses and creators.
  DE  Websites und Automatisierungen für Selbstständige, kleine Unternehmen und Creator.

BOOKING BUTTON (everywhere)
  EN  Book a 30-minute call
  DE  30-Minuten-Erstgespräch buchen

/SERVICES EXPANSION (under the H1)
  EN  You tell me what you need; I build it and take care of the technical side. Impressum, privacy policy, search-engine basics and accessibility are built in from the start. Website and automation come from one person – you talk directly to the one who builds it. Everything runs in your name and belongs to you; if you'd rather not deal with it yourself, I'll look after it.
  DE  Sie sagen, was Sie brauchen – ich baue es und kümmere mich um die Technik. Impressum, Datenschutzerklärung, Suchmaschinen-Grundlagen und Barrierefreiheit sind von Anfang an eingebaut. Website und Automatisierung kommen aus einer Hand – Sie sprechen direkt mit dem, der es baut. Alles läuft in Ihrem Namen und gehört Ihnen; wenn Sie sich nicht selbst darum kümmern möchten, übernehme ich das.

CARD 1
  EN  Website
      Portfolio or business-card site, a site with online booking, a landing page for one offer, or a blog you publish on yourself. You tell me what you want, I build it – legal pages included. Fixed price.
  DE  Website
      Portfolio- oder Visitenkarten-Website, Website mit Online-Buchung, Landingpage für ein Angebot oder ein Blog, den Sie selbst pflegen. Sie sagen, was Sie wollen, ich setze es um – Impressum und Datenschutz inklusive. Festpreis.

CARD 2
  EN  Automation
      Your inbox sorted by topic, action and urgency, with summaries and reply drafts. One video turned into posts and captions for several platforms, scheduled. An assistant that confirms bookings, sends reminders and answers routine questions – for you, not a chatbot for your customers. Enquiries answered and quote drafts prepared. One workflow at a fixed price; more in the call.
  DE  Automatisierung
      Ihr Posteingang sortiert nach Thema, Aufgabe und Dringlichkeit, mit Zusammenfassungen und Antwortentwürfen. Ein Video wird zu Posts und Untertiteln für mehrere Plattformen, zeitlich eingeplant. Ein Assistent, der Termine bestätigt, Erinnerungen verschickt und Routinefragen beantwortet – für Sie, kein Chatbot für Ihre Kundschaft. Anfragen beantwortet und Angebotsentwürfe vorbereitet. Ein Workflow zum Festpreis; mehr im Gespräch.

CARD 3
  EN  Maintenance & Support
      Updates and security, backups, small changes, and a check that your automations still run – with a short monthly note. If something breaks, I look at it within one working day. Monthly, cancel any time, no minimum term. Everything stays yours.
  DE  Wartung & Support
      Updates und Sicherheit, Backups, kleine Änderungen und die Kontrolle, dass Ihre Automatisierungen weiterlaufen – mit einer kurzen monatlichen Notiz. Wenn etwas ausfällt, sehe ich es mir innerhalb eines Werktags an. Monatlich, jederzeit kündbar, keine Mindestlaufzeit. Alles bleibt Ihres.

PRICE LINE (highlighted, under the cards)
  EN  Prices on request – fixed price after a free 30-minute call and a short follow-up.
  DE  Preise auf Anfrage – Festpreis nach einem kostenlosen 30-Minuten-Gespräch und einer kurzen Rücksprache.

HOW A PROJECT RUNS (5 steps — replaces the 4-step block from T4 on home as well)
  EN  1  Free 30-minute call: what you need, what's realistic.
      2  You work out what you want – with my questions and a short to-do list.
      3  Fixed price, in writing, with what's included and what isn't.
      4  Build, with a preview link and a written update every week.
      5  Handover: everything in your name. Maintenance & Support optional.
  DE  1  Kostenloses 30-Minuten-Gespräch: was Sie brauchen, was realistisch ist.
      2  Sie klären für sich, was Sie wollen – mit meinen Fragen und einer kurzen To-do-Liste.
      3  Festpreis, schriftlich, mit dem, was drin ist und was nicht.
      4  Umsetzung mit Vorschau-Link und jede Woche einem schriftlichen Update.
      5  Übergabe: alles in Ihrem Namen. Wartung & Support optional.

ABOUT BLOCK
  DE  Über mich
      Ich baue seit 2025 Websites und Automatisierungen – bisher für eigene Projekte, aktuell die erste Website für einen Kunden. Sie bekommen dafür volle Aufmerksamkeit, direkten Kontakt und einen fairen Einstiegspreis.
  EN  About me
      I've been building websites and automations since 2025 – so far for my own projects, currently the first website for a client. In return, you get full attention, direct contact and a fair entry price.

CONTACT STRIP
  Button as above · obfuscated luiswg@neurogenesis.dev
  EN  I usually reply within one working day.
  DE  Antwort in der Regel innerhalb von 1 Werktag.
```

### Decisions behind the copy (luis, 2026-09-24)

- Target groups in this order: Selbstständige, kleine Unternehmen, Creator — ranked by likelihood to need, benefit from, and contact him. "Local" dropped in both languages. "Independent professionals" kept ("independents" rejected as informal). Plural "automations" kept.
- Positioning suffix „klar gebaut, in Ihrem Namen, ohne Agentur-Overhead" dropped: too long, and it names "Agentur". The how-I-work content moved into the /services expansion.
- Expansion carries: technical side handled (never phrased as "low-maintenance" — devalues the work); legal pages, SEO basics and accessibility "built in from the start" (no "guarantee": rankings can't be guaranteed, accessibility guarantee implies WCAG certification and liability); website + automation from one person; direct contact with the builder; ownership phrased as a bridge into Maintenance & Support, not an exit.
- Cards are summaries; the full range is laid out in the call (clarity, honesty, tailoring — not "trade secrets"). Full catalogue lives in `services/`.
- Website types offered: portfolio/business card, online booking, landing page, blog the client publishes on. **Not yet:** shop, redesign/migration of existing sites — until more experience.
- Automation examples, one per group + inbox for all: inbox (topic/action/urgency, summaries, reply drafts — the running demo); content pipeline (one video → several platforms; platform-specific editing and multi-language captions only in the call, API access to Instagram/TikTok posting is gated); assistant (bookings, reminders, routine questions; "assistant" not "secretary"; differentiated from customer-facing chatbots; needs an AVV for calendar/customer data); enquiries + quote drafts for businesses. **Invoices/document generation: on request in the call only** (GoBD, e-invoicing phase-in 2027–28, OCR errors, no track record). Newsletter is a website feature, not an automation.
- First automation client may be discounted or pro bono for experience/portfolio/proof (Chantal model).
- Maintenance & Support: everything included; small changes **2 h/month** for now (in quote + agreement, not on the page; may change with experience and income); one-working-day response, not same-day; monthly in advance, cancellable to month end, no minimum term (deliberately unlike agencies).
- Pricing: fixed price after call + client homework + short follow-up; internal buffer on the hours estimate, undefined until scope is clear; AI token cost is an internal cost line, never shown. Estimates (Kostenvoranschlag, § 650 BGB) rejected as messier than fixed quotes. Verify with Steuerberater/IHK before the first paid quote.
- Process made five steps on purpose: the client homework is explicit so it's expected, faster, and yields better feedback. Weekly **written** update, no routine calls.
- Home-hero idea "you still don't have one" kept for outreach messages, not for the page.
- Button label without "free" (luis); "30-Minuten-Erstgespräch" accepted at 29 characters.
- Card 2 wording differentiates the assistant from customer-facing chatbots ("for you, not a chatbot for your customers").

### /about page — settled 2026-09-27 (luis + mentor), round two

Designed question by question 2026-09-27 (input parked 2026-09-24). Replaces the current /about entirely ("software engineer", writing-topics list, neuroscience framing all go). Purpose: a referred prospect or social-profile clicker asks "who is this, is he legit?" Rule applied: present tense only, one concrete detail per interest, nothing that turns false if someone asks "can I see?". Blocks the social-profile link until it ships.

**Layout (for the Architect):** banded sections in the site's system; centred headings, paragraph alignment at the Architect's judgement with a screenshot; interim headshot (Luis's current WhatsApp/social profile photo) at the top with name and "Frankfurt am Main"; no link to /services until the FDR-005 gate opens; `/now` and `/uses` 301 → `/about`; changelog out of the nav.

```
HEADER OF THE PAGE
  Headshot (interim) · Luis Gordon · Frankfurt am Main

WHAT I DO
  EN  I build websites and automations for independent professionals, small businesses and creators – as one person, with AI tooling from day one. Self-taught: freeCodeCamp and The Odin Project got me started in 2025; since January 2026 I've been building for real – this site, the automation stack behind it, and currently my first client website. No degree, no team behind me: you talk to the person who builds it.
      Currently: the first client website, and round two of this site.
  DE  Ich baue Websites und Automatisierungen für Selbstständige, kleine Unternehmen und Creator – allein, mit KI-Werkzeugen vom ersten Tag an. Autodidakt: freeCodeCamp und The Odin Project waren 2025 der Einstieg; seit Januar 2026 baue ich richtig – diese Website, die Automatisierungen dahinter und aktuell die erste Website für einen Kunden. Kein Abschluss, kein Team im Hintergrund: Sie sprechen mit dem, der es baut.
      Aktuell: die erste Kunden-Website und Runde zwei dieser Seite.

THE NAME
  EN  Neurogenesis is the birth of new neurons – the brain rebuilding itself as it learns. Neuro also stands for neural networks, the technology behind the AI tools I build with; artificial and biological learning have more in common than it seems. Genesis is creation. Together: learning and building, with AI.
  DE  Neurogenese ist die Entstehung neuer Nervenzellen – das Gehirn baut sich beim Lernen um. Neuro steht auch für neuronale Netze, die Technik hinter den KI-Werkzeugen, mit denen ich baue; künstliches und biologisches Lernen haben mehr gemeinsam, als es scheint. Genesis ist Schöpfung. Zusammen: lernen und bauen, mit KI.

INTERESTS
  EN  Neuroscience, psychology, philosophy
      Languages – German and English natively, French, Italian, learning Japanese
      Music – I sing, accompanying myself on piano and guitar
      Running (5 km a day) and calisthenics
      AI tooling and automation – how one person does what used to take a team
  DE  Neurowissenschaft, Psychologie, Philosophie
      Sprachen – Deutsch und Englisch als Muttersprachen, Französisch, Italienisch, dabei Japanisch zu lernen
      Musik – ich singe und begleite mich selbst am Klavier und an der Gitarre
      Laufen (täglich 5 km) und Calisthenics
      KI-Werkzeuge und Automatisierung – wie eine Person schafft, was früher ein Team brauchte

CONTACT
  EN  Email luiswg@neurogenesis.dev · [Book a 30-minute call] · I usually reply within one working day.
  DE  E-Mail luiswg@neurogenesis.dev · [30-Minuten-Erstgespräch buchen] · Antwort in der Regel innerhalb von 1 Werktag.

PROFILES (name + one line, this order)
  EN  LinkedIn – professional profile
      GitHub – code and configs
      Instagram – the occasional photo
      X – what I read and who I follow
      Facebook – if that's where you are
      Threads – the text side of Instagram
  DE  LinkedIn – berufliches Profil
      GitHub – Code und Konfigurationen
      Instagram – gelegentlich ein Foto
      X – was ich lese und wem ich folge
      Facebook – falls Sie dort unterwegs sind
      Threads – die Textseite von Instagram
```

**Decisions behind it (luis, 2026-09-27):**
- Timeline verified from chat history: 2025 = learning (freeCodeCamp, The Odin Project, exercises); domain bought around New Year 2025/26, neurogenesis.dev + n8n live 2026-01-03 on the Hetzner VPS; Claude Code adopted late Jan 2026. Neither course completed — never claim it; "got me started" only. /services keeps "seit 2025" (technically true; "less than a year" would sound worse).
- Interests: five lines only; more would read try-hard and spoil what later writing can reveal. Past sports and music history (Musik-LK, jazz choir, band) stay off the page; the music line emphasises solo accompanied singing. Japanese without kanji count. No "knack for languages", no self-assessed traits.
- No "show" items yet: no photos/videos of himself online; a music recording is backlog once he has his own space. Interim headshot = current profile photo, consistent across all profiles.
- Socials linked: LinkedIn, GitHub, Instagram, Facebook, X, Threads. **Not linked until content exists:** YouTube, TikTok (TikTok likely never unless a demo-video/short-form pipeline happens). X/Threads kept as contact channels; who he follows on X shows breadth. Facebook/Threads lines are placeholders he may change once those have content.
- Certificates: not for client trust (clients don't recognise them); what reads as legit is Gewerbe, Impressum, clean invoices and **Berufshaftpflicht/IT-Haftpflicht** (~€10–25/month, verify) → Gründung list. Free badges (freeCodeCamp RWD, n8n Level 1/2, Anthropic Academy) only in a slow week, never before Chantal's site.

## Design rules + copy for the site-wide sweep (luis + mentor, 2026-09-27 afternoon, settled)

Discovery session on the Architect's 15:40 handoff. One design philosophy for every page; the Architect writes the design system from this and Claude Code sweeps the site in one pass. Supersedes: navy as a band colour, blue as the newsletter band, the 30-minute button label, the /services About block, the newsletter on home, the T4 home process/about bands, and the "no contact form" line in Contact below.

### Bands — four colours, one meaning each

| Band | Colour | Meaning | Used for |
|---|---|---|---|
| Intro | dark gradient (black→grey) | top of a page | first band on **every** page: H1 + one line. Home: same band, full height, with the button. /about: same band with the headshot, heading left-aligned with its text |
| Content | off-white | browse | default second band: cards, lists, forms, contact details |
| Reading | white | read | article bodies, legal bodies, the /about socials block |
| Footer | dark grey | end | every page |

Navy is **gone** as a band. Rules: dark opens, dark grey closes, off-white is the default second band, white only for text people read, never two light bands adjacent. No fifth colour.

### Accent

- One accent, `#1F6FEB`, with exactly two jobs: **action** (every button, the header Contact link) and **brand** (the `NEUROGENESIS DEV` wordmark and its box). Never a band, never decoration.
- Buttons: accent fill on every band. At most two per page (intro + contact section).
- Button label everywhere: **EN "Book a call" / DE „Gespräch buchen"** (Cal.com shows the durations). "Free, 30 minutes" lives in the text near the button, not on it.

### Page anatomy (every page: intro → content → footer)

| Page | Bands in order |
|---|---|
| Home | intro (name, offer line, button) → What I offer: 3 cards + "All services →" (off-white) → Automation in action: demo tile, one caption, "How it works →" /services (white, dark media) → Projects teaser: 2 tiles, "All projects →" (off-white) → Contact: heading, email, button (white) → footer. **No** newsletter, process or about on home |
| /services | intro (offer line + expansion) → cards → price line → How a project runs → contact → footer. **About block removed.** Demo not repeated |
| /projects | intro → project list (off-white) → footer |
| /articles | intro ("first articles coming" is its line) → newsletter form (off-white) → footer |
| /newsletter | intro → form (off-white) → footer |
| /about | intro with headshot → what I do / name / interests (off-white) → contact incl. socials (white) → footer |
| /contact | intro → contact form + email + button (off-white) → footer |
| /faq | intro → questions (off-white) → footer |
| /impressum, /privacy | intro (line = Stand/Last updated) → body (white) → footer |
| Article page | intro (title, date, reading time; line = summary) → body (white) → newsletter form (off-white) → footer |

Home is the summary page: every section is the short form of another page plus a link there. /services is the full offer, readable alone. Content may repeat across pages; it must not repeat within one.

Newsletter appears only on /newsletter, /articles and article pages.

### Footer (Huberman layout)

Pages list left (bold): Services · Projects · Articles · About. Divider. Columns right: **Contact** = Contact · FAQ · Book a call · Email; **Updates** = Newsletter · Articles. Legal (Impressum · Privacy) and © as fine print. Social icons.

### Contact form (/contact) — Architect scope

Cloudflare Pages Function → Brevo transactional mail to luiswg@neurogenesis.dev (Brevo already a processor). Spam: Cloudflare Turnstile (no Google). Privacy policy: new "Kontaktformular" section DE + EN (data, purpose, retention, Brevo + Turnstile), generator re-run. Fields Name · E-Mail · Nachricht; no attachments.

### Copy — intro lines and modules (EN + DE)

```
HOME        H1 Luis Gordon
            EN  Websites and automations for independent professionals, small businesses and creators.
            DE  Websites und Automatisierungen für Selbstständige, kleine Unternehmen und Creator.
            Button  Book a call / Gespräch buchen
            Cards band heading  What I offer / Was ich anbiete → link  All services → / Alle Leistungen →
            Demo band heading   Automation in action / Automatisierung in Aktion
            Demo band lead      EN  One workflow that runs today: the inbox sorts itself.
                                DE  Ein Workflow, der heute läuft: der Posteingang sortiert sich selbst.
            Demo alt/captions   placeholders until T8; DE alt "Demo: Posteingang wird automatisch sortiert" / "n8n-Workflow"
            Demo link           How it works → / So funktioniert es →  (to /services)
            Projects band       Projects / Projekte → All projects → / Alle Projekte →
            Contact band        Contact / Kontakt · email · button · EN "Free, 30 minutes. I usually reply within one working day." DE „Kostenlos, 30 Minuten. Antwort in der Regel innerhalb von 1 Werktag."

/SERVICES   H1 offer line; line = expansion paragraph (T5). Cards, price line, 5 steps, contact band as on home. DE process heading: So läuft ein Projekt ab.

/PROJECTS   H1 Projects / Projekte
            EN  Own projects so far. Client work follows after launch, with permission.
            DE  Bisher eigene Projekte. Kundenarbeiten folgen nach dem Launch, mit Erlaubnis.

/ARTICLES   H1 Articles / Artikel
            EN  The first articles are in the works. Sign up below and I'll email you when one is out.
            DE  Die ersten Artikel sind in Arbeit. Tragen Sie sich unten ein, ich melde mich, sobald einer erscheint.

/NEWSLETTER H1 Newsletter
            EN  No schedule, no filler: one email when I publish an article or something changes here. English only.
            DE  Kein Rhythmus, kein Füllstoff: eine E-Mail, wenn ein Artikel erscheint oder sich hier etwas ändert. Nur auf Englisch.
            Form heading  EN Get the next article by email   DE Den nächsten Artikel per E-Mail   (replaces "Stay in the loop"; old topics line removed)

/ABOUT      H1 Luis Gordon · Frankfurt am Main (headshot); line = "What I do" paragraph (settled 2026-09-27 morning)
            Interests: the five lines + sixth:
            EN  Self-improvement and biohacking – physical and mental, behavioural and chemical
            DE  Selbstverbesserung und Biohacking – körperlich und mental, verhaltensbezogen und chemisch

/CONTACT    H1 Contact / Kontakt
            EN  A question, a project, or just a hello – write to me or book a call. I usually reply within one working day.
            DE  Eine Frage, ein Projekt oder einfach ein Hallo – schreiben Sie mir oder buchen Sie ein Gespräch. Antwort in der Regel innerhalb von 1 Werktag.
            Form   Name · Email / E-Mail · Message / Nachricht · button Send message / Nachricht senden
            Success EN Thanks, your message is in my inbox. I'll reply within one working day.
                    DE Danke, Ihre Nachricht ist angekommen. Ich antworte innerhalb von 1 Werktag.
            Under the form: email as link · [Book a call] – free, 30 minutes / [Gespräch buchen] – kostenlos, 30 Minuten

/IMPRESSUM  H1 Impressum · line  Stand: <date> / Last updated: <date>
/PRIVACY    H1 Datenschutz / Privacy · line  Stand: <date> / Last updated: <date>

/FAQ        H1 FAQ · EN  Questions people ask before the first call. More will follow from real enquiries.
                     DE  Fragen, die vor dem ersten Gespräch kommen. Weitere folgen aus echten Anfragen.
  1 EN  What does a website cost?  — Prices on request: a fixed price after a free 30-minute call and a short follow-up, once the scope is clear. No hourly surprises.
    DE  Was kostet eine Website?  — Preise auf Anfrage: ein Festpreis nach einem kostenlosen 30-Minuten-Gespräch und einer kurzen Rücksprache, sobald der Umfang klar ist. Keine Stundenzählerei.
  2 EN  How long does it take?  — Usually 6–8 weeks from the written scope, if texts and photos arrive in the first two weeks. Without them, placeholders.
    DE  Wie lange dauert es?  — In der Regel 6–8 Wochen ab dem schriftlichen Umfang, wenn Texte und Fotos in den ersten zwei Wochen kommen. Ohne sie: Platzhalter.
  3 EN  Who owns the site?  — You. Domain, accounts and code are in your name from day one; I'm a member, not the owner. You can leave any time.
    DE  Wem gehört die Website?  — Ihnen. Domain, Konten und Code laufen von Anfang an auf Ihren Namen; ich bin Mitglied, nicht Inhaber. Sie können jederzeit gehen.
  4 EN  Can I edit it myself?  — Text and images, yes; the handover note shows how. If you'd rather not, Maintenance & Support covers it.
    DE  Kann ich sie selbst bearbeiten?  — Texte und Bilder, ja; die Übergabe-Notiz zeigt wie. Wenn Sie das lieber abgeben: Wartung & Support.
  5 EN  What do you need from me?  — A 45-minute call, your answers to a short question list, then texts and photos within two weeks. That's it.
    DE  Was brauchen Sie von mir?  — Ein 45-Minuten-Gespräch, Ihre Antworten auf eine kurze Fragenliste, dann Texte und Fotos innerhalb von zwei Wochen. Mehr nicht.
  6 EN  Do you host the site?  — No. It runs in your own account (Cloudflare Pages), you pay the provider directly, and there's no hosting fee to me.
    DE  Hosten Sie die Website?  — Nein. Sie läuft in Ihrem eigenen Konto (Cloudflare Pages), Sie zahlen den Anbieter direkt, und an mich geht keine Hosting-Gebühr.
```

### Decisions behind it (luis, 2026-09-27)

- Navy dropped as a band: it was reserved for a newsletter that appears twice on the whole site, blocking a colour that could carry meaning site-wide. Blue returns as the single accent for action + brand (Chris Williamson model: one colour means "act").
- Home vs /services: home = summary page, every section a short form plus link; /services = full offer for outreach readers. Redundancy across pages is fine, within a page not. About block off /services (the expansion says who builds it; /about is in the header). Process off home. Newsletter off home ("not what I offer").
- Contact form: wanted despite earlier deferral; built today with Pages Function + Brevo + Turnstile.
- Button: "Book a call" / „Gespräch buchen" — Cal.com shows durations.
- FAQ page now, with six starter questions answered from settled decisions (no invented track record); real enquiries replace/extend them. Reason: footer balance and a visible intent to take client questions seriously.
- Sixth interest line kept abstract on purpose (A over the example-laden B) so it doesn't outweigh the other five; no substance names on a client-facing page.
- Newsletter: notification list, English-only, no cadence promised; bilingual notification mails possible later; a newsletter philosophy is a backlog item once it does more than article/business heads-ups.

## R5 — /about rewrite, home hero, /contact email (luis + mentor, 2026-09-28, settled)

Discovery on the Architect's 2026-09-27 23:30 handoff. Replaces the /about copy of 2026-09-27 morning. Reason (luis): the old text led with weaknesses ("no degree", "as one person", "with AI tooling from day one") and read as a disposable vibe coder; the name section was scientifically loose; the interests read as a list. New stance: self-taught solo developer, no team and none needed, AI as the main tool and an offered field — never the headline; no degree not advertised (inferable on LinkedIn); scientific accuracy in the name section; interests ordered by relevance to the offer.

**Layout (Architect):** all headings on /about left-aligned with their text (incl. socials). Try two variants for screenshots: (1) intro band = headshot + name/location only, the three blocks on off-white; (2) everything in the gradient band, headshot left. Threads removed from the profile list. Design v2's navy booking band stands (approved on staging 2026-09-27) — rule: navy = booking band only, one per page.

**Home hero:** name stays the headline (Chiang precedent; the earlier "offer line as headline" idea was impostor syndrome, luis). Offer line one step larger (smaller ratio to the name); hero ~60–70 % viewport. Offer line text unchanged for now.

**Spelling rule:** American English site-wide (optimization, color, center). Architect to check consistency.

```
/ABOUT

INTRO BAND: headshot · Luis Gordon · Frankfurt am Main

WHAT I DO
EN  I build websites and automations for independent professionals, small businesses and creators – on my own, and by choice. Self-taught: freeCodeCamp's free curriculum got me started in 2025; since January 2026 I've been building for real – this site, the automation stack behind it, and my first client website.
    No team behind me, and none needed. The admin and grunt work that eats a solo developer's week – inbox, drafts, boilerplate – runs on automations I built for myself first. What's left is what you're actually paying for: the thinking, the building, and one person who answers for the result.
DE  Ich baue Websites und Automatisierungen für Selbstständige, kleine Unternehmen und Creator – allein, und mit Absicht. Autodidakt: Der kostenlose Lehrplan von freeCodeCamp war 2025 der Einstieg; seit Januar 2026 baue ich richtig – diese Website, die Automatisierungen dahinter und meine erste Website für einen Kunden.
    Kein Team im Hintergrund, und keins nötig. Die Verwaltungs- und Fleißarbeit, die einem Solo-Entwickler die Woche frisst – Posteingang, Entwürfe, Standardcode – läuft über Automatisierungen, die ich zuerst für mich selbst gebaut habe. Was bleibt, ist das, wofür Sie tatsächlich bezahlen: das Denken, das Bauen und eine Person, die für das Ergebnis geradesteht.

THE NAME
EN  Neurogenesis is the growth of new neurons – in adults mostly in the hippocampus, the part of the brain tied to learning and memory. The prefix also points to neural networks, the technique behind the AI models I use and, in automations, offer: artificial and biological networks share one idea – learning by strengthening the right connections. Genesis is creation. The name is the job description: keep learning, keep creating.
DE  Neurogenese ist das Entstehen neuer Nervenzellen – bei Erwachsenen vor allem im Hippocampus, dem Teil des Gehirns, der für Lernen und Gedächtnis zuständig ist. Die Vorsilbe verweist auch auf neuronale Netze, die Technik hinter den KI-Modellen, die ich nutze und in Automatisierungen anbiete: Künstliche und biologische Netze teilen eine Idee – Lernen, indem die richtigen Verbindungen gestärkt werden. Genesis ist Schöpfung. Der Name ist die Stellenbeschreibung: weiterlernen, weiter erschaffen.

INTERESTS / INTERESSEN  (no intro line)
EN  Automation and AI – how one person does what used to take a team
    Neuroscience, psychology, philosophy – what I read, listen to and watch most
    Languages – German and English natively, French, Italian, learning Japanese
    Optimization – of body and mind (sleep, light, training, supplements, habits) and of everything else I run: hardware, software, routines
    Running (5 km a day) and calisthenics
    Music – I sing, accompanying myself on piano and guitar
DE  Automatisierung und KI – wie eine Person schafft, was früher ein Team brauchte
    Neurowissenschaft, Psychologie, Philosophie – was ich am meisten lese, höre und schaue
    Sprachen – Deutsch und Englisch als Muttersprachen, Französisch, Italienisch, dabei Japanisch zu lernen
    Optimierung – von Körper und Kopf (Schlaf, Licht, Training, Supplemente, Gewohnheiten) und von allem anderen, das ich betreibe: Hardware, Software, Abläufe
    Laufen (täglich 5 km) und Calisthenics
    Musik – ich singe und begleite mich selbst am Klavier und an der Gitarre

PROFILES (this order; Threads back in 2026-09-28 — see marketing/profiles.md)
EN  LinkedIn – professional profile
    GitHub – code and configs
    Instagram – mostly quiet
    X – short thoughts and insights
    Threads – the same short thoughts, for the Instagram crowd
    Facebook – if that's where you are
DE  LinkedIn – berufliches Profil
    GitHub – Code und Konfigurationen
    Instagram – meist still
    X – kurze Gedanken und Einsichten
    Threads – dieselben kurzen Gedanken, für die Instagram-Seite
    Facebook – falls Sie dort unterwegs sind

/CONTACT — email line above the form, in the intro area
EN  Prefer your own mail app? luiswg@neurogenesis.dev – same inbox, same reply time.
DE  Lieber im eigenen Mailprogramm? luiswg@neurogenesis.dev – gleicher Posteingang, gleiche Antwortzeit.
```

Notes: "You talk to the one who builds it" dropped from /about as redundant (luis). Science check on the name section: ANNs are loosely inspired by neurons; the only claim made is the shared idea of learning by adjusting connection strengths. Adult neurogenesis: mostly hippocampal, contested in humans — the sentence says "mostly", nothing more.

### Round two backlog (luis, 2026-09-27) — order agreed

1. **/about** — copy above → Architect. Blocks linking the site in socials.
2. **/now, /uses, changelog** — drop; 301 to /about; changelog out of nav (git history is the changelog). Do not merge into /work.
3. **Articles** — unpublish the AI-written posts (keep as drafts in the repo); /articles becomes a short "first articles coming" note + newsletter form.
4. **/about design** — banded, centred headings; Architect proposes with screenshot.
5. **Profile audit (Luis, ~1 h)** — same photo everywhere, same display name, bio = offer line, link to the site; Facebook: unlink old photos. Checklist from Mentor on request.
6. **GitHub cleanup (Architect)** — pick 3–5 repos to make public (site repo, sanitised n8n exports, dotfiles; Memogenesis only if defensible); names, descriptions, READMEs; **secret-scan full history (gitleaks / GitHub secret scanning) before flipping any repo public**.
7. **Writing pipeline** — last. Keep Obsidian as editor; folder → script copies to repo → push → existing daily rebuild publishes at date. The missing piece is an **authoring guide** (frontmatter fields, callouts, Mermaid, series, what is auto-generated: reading time, ToC) — Architect writes it, only after Luis has written one article by hand. Open: what exactly failed in the previous pipeline decides fix vs rebuild.

## Language split

**Superseded 2026-09-21 by [[../decisions/fdr-007-bilingual-site-with-language-switch|FDR-007]]:** business and legal pages are bilingual (EN at `/`, DE under `/de/`) with a switch; articles EN-only. The German copy above stays the DE version; EN versions follow the same wording rules. Previous rule, kept for history:

DE canonical for /services, /contact, /impressum, /datenschutz. Blog stays EN; EN /privacy is a translation of /datenschutz. No i18n/hreflang at this size.

## Contact (luis, 2026-09-14)

- Obfuscated mailto luiswg@neurogenesis.dev; "Antwort in der Regel innerhalb von 1 Werktag"
- Booking: Cal.com, https://cal.com/luiswg-neurogenesis (account on the personal Google login, Google Calendar connected; username may change — keep the URL as a config value). Link only, no embed. To do: event type "Erstgespräch (30 Min)"
- No contact form (extra infrastructure; deferred)
- Phone: none until the first paid client, then a VoIP number forwarding to the mobile (see [[model]] cost triggers). Impressum stays phone-free

## Status 2026-09-15 (Architect)

/impressum, /datenschutz (generator free licence, hand-edited), /contact with Cal.com button, footer links, 2-click embed consent: live. /services built with the copy above, unlinked + noindex, one commit from live — gate unchanged. Repo renamed neurogenesis-dev. Deploys via GitHub Actions.

## Status 2026-09-19 (Architect handoff)

- GitHub rename Luis-Gordon → neurogenesisdev closed everywhere (remotes, site, workflows, vaults, scripts, MCP configs). Placeholder account `Luis-Gordon` created 2026-09-19 (luiswgordon@gmail.com) so the old URL can't be claimed by a stranger — item closed.
- Cloudflare: new Pages project `neurogenesis-dev`, both domains active; CI uses a Pages-only token (expires 2027-09, reminder due 2027-08); four old tokens deleted.
- Newsletter signup form was missing from the live site from the Actions-deploy switch until 2026-09-17 — fixed, live again. Lesson: [[../lessons/2026-09-17-newsletter-form-silent-outage]].
- Blog pipeline fixed (expired tokens were the "failed daily" cause); timed publishing via daily rebuild; test posts removed. Blogging paused; 1 post/month when resumed.
- Hours logging live: `h 1.5 chantal` (`h -y …` yesterday) → business/hours.csv; weekly triage reports hours, flags > 20 h/week. Convention: log at session end, round down.
- 2026-09-21: [[site-design-brief]] written and active — handed to the Architect for design inventory, TDR addendum (i18n per FDR-007) and the Claude Code task list. Hard cap 15 h. Still open: /services gate per [[../decisions/fdr-005-no-public-prices-until-first-paid-quote|FDR-005]].
- Closed 2026-09-20: DMARC relay — Send-as now via Brevo, `dkim=pass`/`dmarc=pass`, policy `p=quarantine` (FDR-004 status). Also 2026-09-20: Claude phone/desktop chats can read both vaults and drop notes into the inbox repo via the new MCP endpoint (infra-docs ADR-0037). Datenschutz: **Brevo** (EU, transactional email relay) is a new processor to list.

## Newsletter

Kit form stays (account exists: username "Neurogenesis", login on the Proton address — KeePass entry). Double-Opt-in; listed in the Datenschutzerklärung as US transfer / DPF. Kit free plan has no open/click tracking, so the Datenschutz text says "no tracking" — **upgrading Kit changes that section** (Architect, 2026-09-15).

## Case studies

Chantal's site becomes the first /projects case study after launch — only with written consent (a saved WhatsApp message suffices) covering name/brand, screenshots, link, testimonial quote. Consent is part of the pro-bono trade and still to be confirmed with her (see clients/).

## Datenschutzerklärung — processors to list

Cloudflare (Pages hosting, Web Analytics cookieless, Email Routing) · Google Gmail (email storage/processing, US/DPF) · Kit newsletter (US/DPF, DOI) · Cal.com · Brevo (EU, outgoing mail relay, since 2026-09-20) · YouTube-nocookie and X embeds behind consent (2-click) · jsdelivr · fonts self-hosted (confirm no Google Fonts requests). No client-data section until the first care plan holding customer data (then also AVV).
Giscus comments removed from the site 2026-09-25 (no comment function; privacy files updated, T7b).
