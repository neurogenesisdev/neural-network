---
type: area
status: active
created: '2026-04-21T06:32:56Z'
updated: '2026-10-03T18:00:00Z'
tags: [personal, habits, values, vices, self-tracking]
# ── Registry: the daily routine, every step a tracked habit ──────────
# The daily note template and the habit tracker on maps/home.md both read this list.
#   id        permanent key. Never change it. Everything else is free to edit.
#   name      label shown in the daily note and dashboard: one or two words, Title Case.
#   def       definition: what counts as done. Shown on hover in the dashboard.
#   block     routine block the entry belongs to: Wake, Training, Reset, Work, Wind-down.
#             The daily note and the dashboard group by it.
#   optional  true = a suggestion, never "missed": no ✗, no streak, only a count.
#   minutes   true = the checkbox gets a [min:: ] field; the dashboard sums it.
#   schedule  none = every day | days: [Mon, Wed, Fri] | per_week: 5
#   since     first day the rule applies (YYYY-MM-DD).
#   until     optional last day; set it to retire a habit. Retired habits keep their
#             history and show under "Archived" in the dashboard, not in the active view.
#   actions   optional buttons shown next to the habit in the dashboard:
#               { label, template, folder }  creates a new note from a template
#               { label, open }              opens a note (path, or `today`)
# To change a schedule without rewriting history: add a second entry with
# the same id and a later `since`, directly below the first.
# List order = the order of the day = display order.
habits:
  # ── Wake ──
  - id: wake-alarm
    name: Alarm
    def: Out of bed at the first 07:00 alarm, no snooze.
    block: Wake
    since: 2026-10-24
  - id: make-bed
    name: Make Bed
    def: Bed made before leaving the bedroom.
    block: Wake
    since: 2026-10-24
  - id: hydration
    name: Hydration
    def: The 750 ml shaker drunk in one go after the first pee. More water is sipped through the morning.
    block: Wake
    since: 2026-10-24
  - id: powders
    name: Powders
    def: Creatine, two electrolyte tablets and the vitamin C tablet in the shaker.
    block: Wake
    since: 2026-10-24
  - id: morning-hygiene
    name: Morning Hygiene
    def: Tongue scraping, Listerine, brushing (spit, don't rinse), face wash.
    block: Wake
    since: 2026-10-24
  - id: coffee
    name: Coffee
    def: First coffee, 30–45 minutes before the run.
    block: Wake
    since: 2026-10-24
  - id: fuel
    name: Fuel
    def: Pre-workout snack, usually a banana.
    block: Wake
    since: 2026-10-24
  - id: cyclic-hyperventilation
    name: Hyperventilation
    def: One round of cyclic hyperventilation, seated or lying down. Never in or near water.
    block: Wake
    since: 2026-10-24
  # ── Training ──
  - id: isometrics
    name: Isometrics
    def: Antranik's bodyline drill, as the warm-up.
    block: Training
    since: 2026-10-24
  - id: run
    name: Run
    def: 5 km minimum, 10 km maximum. Zone 2 / easy, threshold / PR, intervals, freeform or ruck. Type and distance go in run_type and run_km.
    block: Training
    since: 2026-10-01
  - id: morning-light
    name: Morning Light
    def: 10 minutes or more of outdoor daylight within an hour of sunrise. The run counts. Never look straight at the sun.
    block: Training
    since: 2026-10-24
  - id: strength
    name: Strength
    def: Air squats, push-ups, sit-ups, 2–4 sets of 25 each.
    block: Training
    since: 2026-10-01
  - id: cold-shower
    name: Cold Shower
    def: Shower with at least a few minutes of cold at the end.
    block: Training
    since: 2026-10-24
  - id: meditation
    name: Meditation
    def: One sit of 10 minutes or more; alternate-nostril breathing by default.
    block: Training
    since: 2026-10-01
  # ── Reset ──
  - id: breakfast
    name: Breakfast
    def: First meal, as soon as possible after training.
    block: Reset
    since: 2026-10-24
  - id: capsules
    name: Capsules
    def: Multivitamin, D+K+iodine and omega-3, with breakfast.
    block: Reset
    since: 2026-10-24
  - id: admin
    name: Admin
    def: Email and replies, before deep work.
    block: Reset
    per_week: 5
    since: 2026-10-24
  - id: business
    name: Deep Work
    def: One uninterrupted deep-work block on clients, outreach or neurogenesis.dev.
    block: Reset
    per_week: 5
    since: 2026-10-01
  - id: blog
    name: Writing
    def: Explain one thing as a concept note, or 300 words toward a blog post.
    per_week: 2
    since: 2026-10-01
  - id: blog
    name: Writing
    def: One Pomodoro on writing. Turning a fleeting note into a permanent note counts; so does work on a draft.
    block: Reset
    per_week: 3
    since: 2026-10-24
    actions:
      - label: New post
        template: _templates/blog-post.md
        folder: areas/writing/drafts
      - label: New concept
        template: _templates/permanent-note.md
        folder: notes
  - id: guitar
    name: Guitar
    def: Any session. Minutes in [min::].
    block: Reset
    optional: true
    minutes: true
    since: 2026-10-24
  - id: piano
    name: Piano
    def: Any session. Minutes in [min::].
    block: Reset
    optional: true
    minutes: true
    since: 2026-10-24
  - id: voice
    name: Voice
    def: Vocal exercises or singing. Minutes in [min::].
    block: Reset
    optional: true
    minutes: true
    since: 2026-10-24
  # ── Work ──
  - id: lunch
    name: Lunch
    def: The meal prep (ground beef and rice), before the shift or on the break.
    block: Work
    since: 2026-10-24
  - id: anki
    name: Anki
    def: One Pomodoro (25 min), or every due review cleared, whichever comes first. Commute or break. Japanese first.
    block: Work
    since: 2026-10-24
  - id: journaling
    name: Journal
    def: Plan and Review written in the daily note. The Review fits the commute home.
    block: Work
    since: 2026-10-01
    actions:
      - label: Today
        open: today
  # ── Wind-down ──
  - id: minerals
    name: Minerals
    def: Magnesium glycinate/lysinate and zinc + copper, with the last water.
    block: Wind-down
    since: 2026-10-24
  - id: melatonin
    name: Melatonin
    def: Taken tonight. Optional, so the nights with and without it can be compared.
    block: Wind-down
    optional: true
    since: 2026-10-24
  - id: morning-prep
    name: Morning Prep
    def: Shaker, powders and run clothes set out for tomorrow.
    block: Wind-down
    since: 2026-10-24
  - id: oral-hygiene
    name: Oral Hygiene
    def: Floss, brush, clean tongue, then wash face.
    since: 2026-10-01
  - id: oral-hygiene
    name: Evening Hygiene
    def: Floss (picks on the couch count), brushing, tongue scraping, face wash.
    block: Wind-down
    since: 2026-10-24
  - id: nose-strip
    name: Nose Strip
    def: Strip on, after the face wash.
    block: Wind-down
    since: 2026-10-24
  - id: reading
    name: Reading
    def: 20 minutes of a book.
    since: 2026-10-01
  - id: reading
    name: Reading
    def: Any reading session, in bed or on the commute. Minutes in [min::].
    block: Wind-down
    optional: true
    minutes: true
    since: 2026-10-24
  - id: physiological-sigh
    name: Physiological Sigh
    def: Three or more in bed, before the mouth tape.
    block: Wind-down
    since: 2026-10-24
  - id: mouth-tape
    name: Mouth Tape
    def: Taped for the night. Nothing more to eat or drink.
    block: Wind-down
    since: 2026-10-24
  - id: earplugs
    name: Earplugs
    def: Earplugs or ear wax in.
    block: Wind-down
    since: 2026-10-24
  - id: sleep-mask
    name: Sleep Mask
    def: Mask on, lights out.
    block: Wind-down
    since: 2026-10-24
  # ── Archived ──
  - id: japanese
    name: Japanese
    def: Retired 2026-10-23; merged into Anki.
    since: 2026-10-01
    until: 2026-10-23
  - id: memorise
    name: Memorise
    def: Retired 2026-10-23; merged into Anki.
    since: 2026-10-01
    until: 2026-10-23
  - id: music
    name: Music
    def: Retired 2026-10-23; split into Guitar, Piano and Voice.
    per_week: 3
    since: 2026-10-01
    until: 2026-10-23
  - id: sleep-hygiene
    name: Sleep Hygiene
    def: Retired 2026-10-23; split into Nose Strip, Mouth Tape, Earplugs and Sleep Mask.
    since: 2026-10-01
    until: 2026-10-23
  - id: coding-practice
    name: Coding
    def: Retired 2026-10-23.
    per_week: 5
    since: 2026-10-01
    until: 2026-10-23
  - id: reach-out
    name: Reach Out
    def: Retired 2026-10-23.
    per_week: 1
    since: 2026-10-01
    until: 2026-10-23
  - id: evening-stack
    name: Evening Stack
    def: Retired 2026-10-03; split into Oral Hygiene and Sleep Hygiene.
    since: 2026-10-02
    until: 2026-10-02
---

# Habits

> [!NOTE] Summary
> Two things live here. The **registry** (the `habits:` list in this note's
> properties) is the daily routine in order, and every step in it is a habit
> that the daily note and dashboard check off. The **catalog** below is
> everything else worth considering. Add a step by adding it to the registry;
> retire one by giving it an `until:` date. Edit the registry in source mode.

## Daily routine (2026-10-24)

The order is the order of a good late-shift day. The registry `def:` is the source; this table mirrors it.

| # | Block | Habit | id | Done when | Schedule |
|---|---|---|---|---|---|
| 1 | Wake | Alarm | wake-alarm | Out of bed at the first 07:00 alarm, no snooze. | daily |
| 2 | Wake | Make Bed | make-bed | Bed made before leaving the bedroom. | daily |
| 3 | Wake | Hydration | hydration | The 750 ml shaker drunk in one go after the first pee. More water is sipped through the morning. | daily |
| 4 | Wake | Powders | powders | Creatine, two electrolyte tablets and the vitamin C tablet in the shaker. Contents: [[supplement-stack]]. | daily |
| 5 | Wake | Morning Hygiene | morning-hygiene | Tongue scraping, Listerine, brushing (spit, don't rinse), face wash. | daily |
| 6 | Wake | Coffee | coffee | First coffee, 30–45 minutes before the run. | daily |
| 7 | Wake | Fuel | fuel | Pre-workout snack, usually a banana. | daily |
| 8 | Wake | Hyperventilation | cyclic-hyperventilation | One round of cyclic hyperventilation, seated or lying down. Never in or near water. | daily |
| 9 | Training | Isometrics | isometrics | Antranik's bodyline drill, as the warm-up. | daily |
| 10 | Training | Run | run | 5 km minimum, 10 km maximum: zone 2 / easy, threshold / PR, intervals, freeform or ruck. Type and distance in `run_type` and `run_km`. | daily |
| 11 | Training | Morning Light | morning-light | 10 minutes or more of outdoor daylight within an hour of sunrise. The run counts. | daily |
| 12 | Training | Strength | strength | Air squats, push-ups, sit-ups, 2–4 sets of 25 each. | daily |
| 13 | Training | Cold Shower | cold-shower | Shower with at least a few minutes of cold at the end. | daily |
| 14 | Training | Meditation | meditation | One sit of 10 minutes or more; alternate-nostril breathing by default. | daily |
| 15 | Reset | Breakfast | breakfast | First meal, as soon as possible after training. | daily |
| 16 | Reset | Capsules | capsules | Multivitamin, D+K+iodine and omega-3, with breakfast. | daily |
| 17 | Reset | Admin | admin | Email and replies, before deep work. | 5x a week |
| 18 | Reset | Deep Work | business | One uninterrupted deep-work block on clients, outreach or neurogenesis.dev. | 5x a week |
| 19 | Reset | Writing | blog | One Pomodoro on writing; fleeting → permanent note counts, so does a draft. | 3x a week |
| 20 | Reset | Guitar | guitar | Any session, minutes logged. | optional |
| 21 | Reset | Piano | piano | Any session, minutes logged. | optional |
| 22 | Reset | Voice | voice | Vocal exercises or singing, minutes logged. | optional |
| 23 | Work | Lunch | lunch | The meal prep, before the shift or on the break. | daily |
| 24 | Work | Anki | anki | One Pomodoro, or every due review cleared. Commute or break. | daily |
| 25 | Work | Journal | journaling | Plan and Review written. The Review fits the commute home. | daily |
| 26 | Wind-down | Minerals | minerals | Magnesium and zinc + copper, with the last water. | daily |
| 27 | Wind-down | Melatonin | melatonin | Taken tonight. | optional |
| 28 | Wind-down | Morning Prep | morning-prep | Shaker, powders and run clothes set out for tomorrow. | daily |
| 29 | Wind-down | Evening Hygiene | oral-hygiene | Floss, brushing, tongue scraping, face wash. | daily |
| 30 | Wind-down | Nose Strip | nose-strip | Strip on, after the face wash. | daily |
| 31 | Wind-down | Reading | reading | Any reading session, in bed or on the commute, minutes logged. | optional |
| 32 | Wind-down | Physiological Sigh | physiological-sigh | Three or more in bed, before the mouth tape. | daily |
| 33 | Wind-down | Mouth Tape | mouth-tape | Taped for the night. Nothing more to eat or drink. | daily |
| 34 | Wind-down | Earplugs | earplugs | Earplugs or ear wax in. | daily |
| 35 | Wind-down | Sleep Mask | sleep-mask | Mask on, lights out. | daily |

Mark a day `- [-]` (sick, injured, travelling) to skip it without breaking the streak.

Context and states (shift, wake and bed time, sleep, mood, energy, focus, run details, training order, meals, late work) are properties of the daily note, not habits. The dashboard writes `logged` whenever a habit is ticked from it, so a `logged` date later than the note's own date shows a retroactive fill.

Trial period: run this set for a few weeks, then trim, merge or reorder at the monthly review. A step that never gets ticked is the first to prune.

## Vices

Habits to avoid or minimize:

- Porn
- Alcohol
- Gaming
- Hard Drugs
- Cannabis

## Catalog

Not tracked. Pick from here when there is room.

### Body

- Run 10 km (stretch goal for the daily run)
- Full calisthenics programme (replaces the bodyweight circuit once running load has settled; keep id `strength`)
- Mobility — 10 minutes of stretching, counterweight to daily running
- Walk — 30 minutes, no headphones
- Lights Out — in bed by a fixed time (not realistic with late shifts)
- Cardio (Zone 2) beyond the daily run
- HIIT
- Martial Arts
- Stretching (AM)
- Deep Stretching (PM)
- Neck Training
- Kegel Training
- Sunlight Viewing (PM)

### Health basics

- Skin Care beyond washing the face
- Minoxidil (AM / PM)

### Mind

- Screen Curfew — no phone in the last hour (not realistic with late shifts)
- Second meditation session
- Past & Future Authoring (weekly)

### Skills

- Notebook — one idea or observation captured per day (currently: a line in the daily Log)
- The Odin Project — parked in favour of LeetCode / freeCodeCamp
- Japanese speaking practice with a native — later; reading comes first
- Language immersion (shows, YouTube, grammar book, literature) — counted in the weekly review, not daily
- Typing Practice
- Chess Practice
- Audiobook

### Creative

- Word Quota — a fixed daily word count (currently folded into Writing)
- Music Production
- Coding Project (personal, outside the business)
- Dancing at a Club

### Folded into active habits

- Deep Work → Business (shown as "Deep Work")
- Explain → Writing
- Plan, Review → Journal
- Japanese, Memorise, general-knowledge flashcards → Anki
- Vocal Training, singing → Voice
- Electrolytes, Supplements (AM / PM) → Powders, Capsules, Minerals
- Oral Hygiene (AM) → Morning Hygiene
- Next Day Preparation → Morning Prep

### Archived (were tracked, now retired)

These stay in the registry with an `until:` date, so their history is kept. The dashboard shows them under "Archived".

- Evening Stack (`evening-stack`) — 2026-10-02 only; split into Oral Hygiene and Sleep Hygiene.
- Japanese (`japanese`), Memorise (`memorise`) — until 2026-10-23; merged into Anki.
- Music (`music`) — until 2026-10-23; split into Guitar, Piano and Voice.
- Sleep Hygiene (`sleep-hygiene`) — until 2026-10-23; split into Nose Strip, Mouth Tape, Earplugs and Sleep Mask.
- Coding (`coding-practice`), Reach Out (`reach-out`) — until 2026-10-23.

### Retired

- University Courses, University Study, University Assignments — not currently enrolled.
