---
type: reference
status: active
created: 2026-01-24
updated: '2026-04-21T10:11:50Z'
tags: [system, documentation, vault, pkm]
---

# Neural Network Vault Documentation

> Personal knowledge management vault using a PARA + MOC + Zettelkasten hybrid approach with Claude Code-assisted linking.

---

## Table of Contents

1. [Overview](#overview)
2. [Design Philosophy](#design-philosophy)
3. [Folder Structure](#folder-structure)
4. [Note Types & Templates](#note-types--templates)
5. [Frontmatter Standards](#frontmatter-standards)
6. [Link Allowlist System](#link-allowlist-system)
7. [MOC Structure](#moc-structure)
8. [Installed Plugins](#installed-plugins)
9. [Weekly Workflow](#weekly-workflow)
10. [Future Plans](#future-plans)
11. [Maintenance Guidelines](#maintenance-guidelines)

---

## Overview

This Obsidian vault ("Neural Network") is a personal knowledge management system designed for:

- **Daily journaling** with comprehensive habit tracking and health metrics
- **Learning & research** with atomic permanent notes that compound over time
- **Project management** for active work with defined end states
- **Reference storage** for documentation, procedures, and knowledge

The system combines three proven PKM methodologies:

| Methodology | What It Provides |
|-------------|------------------|
| **PARA** | Folder structure based on actionability (Projects, Areas, Resources, Archive) |
| **MOCs** | Flexible navigation through linked Maps of Content |
| **Zettelkasten** | Atomic permanent notes with rich cross-linking |

---

## Design Philosophy

### Core Principles

1. **Folders manage workflow stages, not topics**
   - Notes are organized by *what you do with them* (capture, process, reference, archive)
   - Topics emerge through links and tags, not folder hierarchies

2. **Links and MOCs handle conceptual organization**
   - A single note can belong to multiple conceptual categories via links
   - MOCs serve as navigational hubs when note clusters reach 6+ notes

3. **Frictionless mobile capture → weekly batch processing**
   - All mobile captures go to `_inbox/` without any organization overhead
   - Weekly processing sessions with Claude Code propose structure and links

4. **Claude Code-assisted linking with allowlist system**
   - Approved keyword → wikilink mappings stored in `_meta/link-allowlist.md`
   - Once a mapping is approved, future occurrences can be auto-linked

### Why This Structure?

| Problem | Solution |
|---------|----------|
| "Where does this note go?" decision fatigue | Everything goes to `_inbox/`, decide later |
| Notes stay isolated, no emergent insights | Mandatory linking via allowlist system |
| Maintenance burden too high | Weekly batch processing instead of constant upkeep |
| Can't find things | MOCs + search + Dataview queries |

---

## Folder Structure

```
neural-network/
├── _inbox/                    # All capture (fleeting notes, drafts, imports)
│   └── n8n/                   # n8n service ingress (untrusted input)
├── _templates/                # Templater skeletons (one index: _templates/_index.md)
├── _attachments/              # Images, PDFs, media; letters/ holds correspondence scans
├── _generated/                # Machine output only (protected; never edit)
├── _meta/                     # Vault documentation, schema, vocabulary, decisions/, legacy-rules/, scratch/
├── journal/
│   ├── daily/                 # Daily notes: YYYY/MM/YYYY-MM-DD.md
│   └── weekly/ … yearly/      # Reviews: weekly/ monthly/ quarterly/ yearly/
├── notes/                     # Atomic permanent notes (Zettelkasten)
├── projects/                  # Active work with defined end states
├── areas/                     # Ongoing responsibilities (no end date)
│   ├── admin/                 # Cases and correspondence
│   ├── finance/               # Income, recurring, debts, budget
│   ├── health/                # Fasting, exercise, sleep, biohacking
│   ├── career/                # Professional development
│   ├── self/                  # Typology, goals, philosophy, habits; learning/ and experiments/
│   ├── writing/               # Blog: drafts/ and published/
│   ├── infrastructure/        # Machines (by device)
│   └── freelance/             # Freelance practice
├── resources/                 # Reference material by topic (programming, psychology, poetry, prompts, work, …)
├── maps/                      # MOCs for cross-domain navigation; home.md is the dashboard
└── archive/                   # Completed projects, legacy content
```

### Folder Purposes

| Folder | Purpose | When to Use |
|--------|---------|-------------|
| `_inbox/` | Frictionless capture zone | Mobile captures, quick thoughts, drafts, unprocessed imports |
| `journal/daily/` | Daily notes | One per day at `YYYY/MM/YYYY-MM-DD.md`; habit tracking, journaling, work logs |
| `notes/` | Atomic ideas | Permanent notes, one idea per note |
| `projects/` | Active work with deadlines | Has a defined "done" state, time-bound |
| `areas/` | Ongoing responsibilities | No end date, standards to maintain |
| `resources/` | Reference material | Lookup information, doesn't change often |
| `resources/work/` | Work reference material | Procedures, checklists, job-specific info |
| `maps/` | Navigation hubs | MOCs linking related notes |
| `archive/` | Inactive content | Completed projects, deprecated content |
| `_meta/` | Configuration & docs | Link allowlist, vault documentation, schema, vocabulary, PKM reference |
| `_templates/` | Note templates | Templater skeletons for every note type |
| `_attachments/` | Media | Images, PDFs; `letters/` for correspondence scans |

---

## Note Types & Templates

All templates are in `_templates/` and use Templater syntax.

### 1. Fleeting Note

**Purpose:** Quick captures, meeting thoughts, ideas to process later.
**Process within:** 24-48 hours

```yaml
---
created: {{date:YYYY-MM-DD}}
type: fleeting
tags: []
---
```

**Sections:** Capture, Source, Next (checkbox to process or discard)

### 2. Permanent Note

**Purpose:** Single atomic idea in your own words. The building blocks of knowledge.
**Location:** `notes/`

```yaml
---
created: {{date:YYYY-MM-DD}}
type: permanent
tags: []
aliases: []
up: "[[]]"           # Parent MOC
---
```

**Sections:** Idea (one atomic concept), Evidence (sources), Related (links to other notes)

**Rules:**
- One idea per note
- Written in your own words (not copied)
- Must link to at least one other note
- Should connect to a parent MOC via `up:` field

### 3. Literature Note

**Purpose:** Notes on a single source (book, article, paper, video).
**Workflow:** Creates permanent notes from key ideas.

```yaml
---
created: {{date:YYYY-MM-DD}}
type: literature
author:
source:
status: unread      # unread | reading | processed
tags: []
---
```

**Sections:** Summary, Key Ideas (numbered), Quotes, Permanent Notes to Create (checkboxes), Related

### 4. Project Note

**Purpose:** Active work with a defined end state and deadline.
**Location:** `projects/`

```yaml
---
created: {{date:YYYY-MM-DD}}
type: project
status: active      # active | paused | completed
area: "[[]]"        # Parent area (e.g., [[areas/career/_index]])
due:                # YYYY-MM-DD
tags: [project]
---
```

**Sections:** Goal, Success Criteria, Tasks (Next Actions, Waiting For), Resources, Log (dated entries)

### 5. MOC (Map of Content)

**Purpose:** Navigation hub linking related notes on a topic.
**Location:** `maps/`

```yaml
---
created: {{date:YYYY-MM-DD}}
type: moc
tags: [moc]
up: "[[Home]]"      # Parent MOC
---
```

**Sections:** Purpose callout, Core Concepts, Related Ideas, Questions to Explore, Resources, Related MOCs

### 6. Daily Note

**Purpose:** Daily journaling with habit tracking and health metrics.
**Location:** `journal/daily/YYYY/MM/YYYY-MM-DD.md`

See [Frontmatter Standards](#daily-note-schema) for the full schema.

---

## Frontmatter Standards

### Required Fields (All Notes)

```yaml
---
created: YYYY-MM-DD
type: fleeting | permanent | literature | project | daily | moc
tags: []
---
```

### Type-Specific Fields

| Note Type | Additional Fields |
|-----------|-------------------|
| `permanent` | `aliases`, `up` (parent MOC) |
| `literature` | `author`, `source`, `status` |
| `project` | `status`, `area`, `due` |
| `moc` | `up` (parent MOC) |
| `daily` | See Daily Note Schema below |

### Daily Note Schema

The daily note template includes comprehensive tracking for health optimization:

```yaml
---
date: YYYY-MM-DD
creation_time: HH:MM
tags:
  - daily-note
work_shift:            # late / early / off
wake_time:             # HH:MM
bedtime:               # HH:MM, the night before
total_sleep_duration:  # hh:mm format
rem_sleep_duration:    # hh:mm format
deep_sleep_duration:   # hh:mm format
mood_score:            # 1-5 scale
energy_score:          # 1-5 scale
focus_score:           # 1-5 scale
run_type:              # easy / threshold / intervals / freeform / ruck
run_km:                # e.g. 7.5
training_order:        # e.g. iso-run-strength
first_meal:            # HH:MM
last_meal:             # HH:MM
late_work:             # true / false: freelance work after the shift
logged:                # set by the home dashboard when a habit is ticked there
focus_theme:           # Short one-liner (e.g., "Deep Work - Coding")
---
```

An empty property means "not logged", not "no". Notes created before 2026-10-24 also carry `day_category` and `fasting_window`; the template no longer sets them.

**Habit Tracking Format:**

Since 2026-10-02 habits are defined once, in the registry (the `habits:` list in the properties of `areas/self/habits-list.md`): id, name, schedule, start date. The daily note template generates the day's checkboxes from it, and the habit tracker on `maps/home.md` (Datacore) joins the two by id:

```markdown
- [ ] Habit Name #habit [h::habit-id]
```

- `#habit` - Required tag for dashboard detection
- `[h::id]` - Permanent key linking the checkbox to the registry; the label is free text
- `- [-]` - Planned skip (sick, injured, travelling); does not break the streak
- Schedules: every day (default), `days: [Mon, Wed, Fri]`, or `per_week: N`
- `optional: true` - Never counted as missed; no streak, only a count
- `minutes: true` - Adds a `[min:: ]` field to the checkbox; the dashboard sums it per week or month
- `block:` - Routine block (Wake, Training, Reset, Work, Wind-down); the daily note and dashboard group by it
- `logged` (daily note property) - Set to the local date and time whenever a habit is ticked from the dashboard
- Daily notes before 2026-10-02 use the old `[seq::] [frequency::]` format and are not read by the dashboard

---

## Link Allowlist System

### Location

`_meta/link-allowlist.md`

### Purpose

Stores approved keyword → wikilink mappings for Claude Code-assisted linking. When processing inbox notes, Claude Code:

1. Scans note content for keywords in the allowlist
2. Proposes wikilinks for matching keywords
3. Suggests new keywords not yet in the allowlist
4. User approves/rejects each suggestion
5. Approved new keywords are added to the allowlist

### Format

```markdown
## Concepts
- BDNF → [[BDNF]]
- brain-derived neurotrophic factor → [[BDNF]]
- neurogenesis → [[Neurogenesis]]
- intermittent fasting → [[Intermittent Fasting]]
- IF → [[Intermittent Fasting]]

## People
- [Add as encountered]

## Projects
- [Add active project names]

## Topics
- [Add as MOCs are created]
```

### Matching Rules

- Case-insensitive matching
- Multiple keywords can map to the same target note
- Aliases in note frontmatter should also be added here

---

## MOC Structure

### Purpose of maps/

Maps of Content (MOCs) serve as navigational hubs that link related notes across domains. Unlike folders, a note can appear in multiple MOCs.

### When to Create an MOC

Create an MOC when you experience **"mental squeeze"** - typically when you have 6+ notes on a topic and feel overwhelmed finding things.

### Current MOCs

| MOC | Purpose |
|-----|---------|
| `Home.md` | Main dashboard, habit tracker, experiment log, navigation |
| `Neuroplasticity MOC.md` | Brain plasticity, cognitive enhancement, BDNF, neurogenesis |
| `Biohacking MOC.md` | Self-optimization, health protocols, fasting, sleep |

### MOC Hierarchy

```
Home
├── Neuroplasticity MOC
│   └── Concept notes: BDNF, Neurogenesis, Neuroplasticity
├── Biohacking MOC
│   └── Concept notes: Intermittent Fasting
└── [Future MOCs]
```

### Concept Note → MOC Linking

Every permanent note in `notes/` should have an `up:` field pointing to its parent MOC:

```yaml
up: "[[Neuroplasticity MOC]]"
```

---

## Installed Plugins

The following plugins are installed in `.obsidian/plugins/`:

### Core Workflow

| Plugin | Purpose |
|--------|---------|
| **calendar** | Visual calendar for navigating daily notes |
| **periodic-notes** | Daily, weekly, monthly note templates and automation |
| **templater-obsidian** | Advanced templates with logic, date manipulation |
| **dataview** | Query notes as a database, power the habit tracker |
| **obsidian-tasks-plugin** | Task management with due dates, recurrence |

### Capture & Sync

| Plugin | Purpose |
|--------|---------|
| **quickadd** | Fast note capture with templates |
| **whisper** | Speech-to-text transcription for voice notes |
| **todoist-sync-plugin** | Sync tasks with Todoist |

### Visualization

| Plugin | Purpose |
|--------|---------|
| **heatmap-calendar** | Visualize habit streaks as a heatmap |
| **plot-vectors-graphs** | Mathematical plotting (for studies) |
| **obsidian-latex** | LaTeX rendering for equations |
| **obsidian-latex-environments** | Extended LaTeX environment support |

### Integration

| Plugin | Purpose |
|--------|---------|
| **morgen-obsidian** | Calendar integration with Morgen |
| **nexus** | AI-assisted note connections |
| **copilot** | AI copilot for writing assistance |

---

## Weekly Workflow

### Step 1: Review Inbox (10 min)

1. Open `_inbox/` folder
2. For each note:
   - If actionable → Move to `projects/` or create task
   - If reference → Move to appropriate `resources/` subfolder
   - If idea → Process into permanent note or discard
   - If unclear → Move to `_inbox/`

### Step 2: Process Triage (5 min)

1. Open `_inbox/`
2. For each stub:
   - Expand into full note, or
   - Delete if no longer relevant

### Step 3: Claude Code Linking Session (10 min)

Run with Claude Code:

```
Scan _inbox/ and recently modified notes. For each:
1. Check content against _meta/link-allowlist.md
2. Propose wikilinks for matching keywords
3. Suggest new keywords to add to allowlist
4. Propose target folder based on content type
Present changes for batch approval.
```

### Step 4: Review and Approve

1. Review proposed links
2. Approve/reject each suggestion
3. New approved keywords automatically added to allowlist

### Step 5: Update MOCs (if needed)

If new concept notes were created:
1. Add them to relevant MOC
2. Update `up:` field in the concept note
3. Consider creating new MOC if cluster reaches 6+ notes

---

## Future Plans

### ~~Phase 2: Work/Journal Consolidation~~ ✅ Completed (2026-01-25)

Work logs have been consolidated into daily notes:
- 127 work entries migrated to `Journal/Daily/` notes under `## Work` section
- Work reference notes moved to `Resources/Work/`
- Original work logs archived to `Archive/Work Journal Legacy/`
- Daily note template updated with `## Work` section

### Potential Integrations

| Integration | Purpose | Status |
|-------------|---------|--------|
| **Whoop** | Auto-populate sleep metrics (HRV, strain, recovery) | Planned |
| **Cronometer** | Auto-populate nutrition data, fasting windows | Planned |
| **FitNotes** | Auto-populate workout logs | Planned |

These would automate YAML frontmatter population in daily notes.

### Knowledge Expansion

As knowledge grows:
- Create domain-specific MOCs (Nutrition MOC, Sleep MOC, Programming MOC)
- Expand `notes/` with new atomic notes
- Build richer cross-domain connections

---

## Maintenance Guidelines

### Daily (2 min)

- Create daily note from template
- Capture thoughts to `_inbox/` as they arise
- Complete habit checkboxes throughout the day

### Weekly (30 min)

- Process `_inbox/` and `_inbox/`
- Run Claude Code linking session
- Review and update allowlist
- Check `maps/home.md` dashboard

### Monthly (60 min)

- Archive completed projects → `archive/`
- Review Areas for relevance
- Identify note clusters that need MOCs
- Clean up unused tags
- Review and prune allowlist

### When to Create New MOCs

Create an MOC when:
- You have 6+ notes on a topic
- You feel "mental squeeze" finding related notes
- Cross-domain connections are emerging (e.g., fasting ↔ meditation ↔ neuroplasticity)

### When to Archive

Archive when:
- A project is completed
- An area is no longer relevant
- Content is outdated but worth preserving

**Never delete** - always archive to `archive/` for future reference.

### Health Checks

Periodically verify:
- [ ] No orphan notes in `projects/`, `areas/`, `resources/` (all should link somewhere)
- [ ] All concept notes have `up:` field pointing to an MOC
- [ ] `_inbox/` is processed weekly (not accumulating indefinitely)
- [ ] Allowlist is growing with new approved keywords
- [ ] MOCs are not stale (update when adding new concept notes)

---

## Quick Reference

### File Naming

- **Daily notes:** `YYYY-MM-DD.md`
- **Concept notes:** Descriptive title, no date prefix (e.g., `BDNF.md`)
- **MOCs:** `Topic MOC.md` (e.g., `Neuroplasticity MOC.md`)
- **Projects:** Descriptive title (e.g., `Build SaaS MVP.md`)

### Tag Conventions

| Tag | Purpose |
|-----|---------|
| `#daily-note` | Daily journal entries |
| `#habit` | Habit tracking items |
| `#moc` | Maps of Content |
| `#project` | Project notes |
| `[topic tags]` | Content categorization |

### Keyboard Shortcuts (Recommended)

| Action | Suggested Shortcut |
|--------|-------------------|
| Open daily note | `Ctrl/Cmd + D` |
| Quick capture to inbox | `Ctrl/Cmd + Shift + N` |
| Open command palette | `Ctrl/Cmd + P` |
| Search vault | `Ctrl/Cmd + Shift + F` |
| Open Home MOC | `Ctrl/Cmd + H` (custom) |

---

*Last updated: 2026-01-25*
*Documentation generated with Claude Code assistance*
