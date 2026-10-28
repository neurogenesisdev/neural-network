---
type: reference
status: active
created: 2026-01-24
updated: '2026-04-21T10:11:50Z'
tags: [system, reference, pkm, methodology, obsidian]
---

# Personal Knowledge Management with Obsidian: Complete Reference

Obsidian stores notes as plain Markdown files with bi-directional linking and an extensible plugin ecosystem. The most effective PKM systems combine a clear organizational methodology (Zettelkasten, PARA, MOCs, Johnny Decimal), consistent daily habits, and carefully selected plugins. **The universal principle across all methodologies: folders manage workflow stages and note types, not topics. Links and tags handle conceptual organization.**

---

## PKM Methodologies

### Zettelkasten

#### Overview

The Zettelkasten method, pioneered by sociologist Niklas Luhmann (70+ books using his slip-box), builds knowledge through three note types:

| Note Type | Purpose | Processing Time |
|-----------|---------|-----------------|
| **Fleeting notes** | Quick captures, meeting thoughts | Process within 24-48 hours |
| **Literature notes** | One note per source with bibliographic references | During/after reading |
| **Permanent notes** | Single atomic ideas in your own words, heavily linked | From fleeting/literature notes |

The power comes from connections—every new permanent note should link to at least one existing note, creating emergent structure over time.

**Best for:** Researchers, academics, writers who need ideas to compound over years. Long-term knowledge development over immediate productivity.

**Core practices:**
- Write in your own words rather than copying
- Limit each note to one atomic idea
- Process fleeting notes daily before context is lost
- Every permanent note links to at least one existing note

**Common mistakes:**
- Creating orphan notes without links
- Making notes too long with multiple ideas
- Trying to perfect notes before creating them

#### Zettelkasten Folder Structures

**Gröpl's Starter Kit structure** (1,400+ GitHub stars, implements Sönke Ahrens' workflow):

```
📁 0_Inbox/              ← Fleeting notes land here first
📁 1_Fleeting Notes/     ← Quick captures, meeting thoughts
📁 2_Literature Notes/   ← One note per source (book, article, paper)
📁 3_Permanent Notes/    ← Atomic ideas in your own words (Zettels)
📁 4_Reference Notes/    ← People, terms, tools, quotes
📁 5_Structure Notes/    ← MOCs, indices, project outlines
📁 Assets/               ← Images, attachments
📁 Templates/            ← All note templates
```

Notes flow from Inbox → Fleeting → Literature → Permanent, with each folder representing a level of processing.

**Purist flat approach** (zettelkasten.de community):

```
📁 Fleeting/
📁 Permanent/
📁 Templates/
📁 Attachments/
├── (All permanent Zettels live at root level)
```

Trusts Obsidian's linking and search rather than folder hierarchies. The zettelkasten.de consensus: for digital systems, **timestamp UIDs in filenames are largely unnecessary** since Obsidian handles link updates automatically. Store creation dates in YAML frontmatter instead.

#### Zettelkasten File Naming Conventions

| Convention | Example | Best For |
|-----------|---------|----------|
| **Timestamp UID + Title** | `202401241430 Atomic notes.md` | Unique, chronologically sortable |
| **Date prefix** | `2024-01-24_Atomic notes.md` | Simpler chronological sorting |
| **Title only** | `Atomic notes.md` | Clean, readable (recommended for digital) |
| **Luhmann alphanumeric** | `1a1 Subtopic.md` | Traditional hierarchical relationships |

#### Zettelkasten Note Templates

**Fleeting note template:**
```markdown
---
created: {{date:YYYY-MM-DD}}T{{time:HH:mm}}
type: fleeting
status: inbox
tags: [fleeting]
---

# {{title}}

## Idea


## Source
- Where did this come from?

## Next Steps
- [ ] Process into permanent note
```

**Literature note template:**
```markdown
---
title: "{{title}}"
tags: [literature-note]
author: 
year: 
source:
status: unread
created: <% tp.file.creation_date("YYYY-MM-DD") %>
---

# {{title}}

## Metadata
- **Author(s)**: 
- **Year**: 
- **Source**: 

## Summary
_Brief summary in your own words_

## Key Ideas
1. 
2. 
3. 

## Quotes
> "Quote here" (p. XX)

## Permanent Notes to Create
- [ ] 

## Related
- [[]]
```

**Permanent note (Zettel) template:**
```markdown
---
created: {{date:YYYY-MM-DD}}T{{time:HH:mm}}
type: permanent
status: evergreen
tags: [permanent]
aliases: []
---

# {{title}}

## Idea
(One atomic idea in your own words)



## Evidence
- [[Source 1]]

## Related
- [[Related Zettel 1]]
- [[Related Zettel 2]]

## Questions
- 

---
*Links to:* [[MOC Name]]
```

#### Zettelkasten Essential Plugins

Dataview, Templater, Update Time on Edit, Book Search, Unique Note Creator (for timestamp-based UIDs if desired)

---

### PARA Method

#### Overview

Tiago Forte's PARA method sorts everything into four folders based on **actionability**, not topic:

| Folder | Contents | Examples |
|--------|----------|----------|
| **Projects** | Short-term efforts with deadlines and specific outcomes | "Complete website redesign," "Write Q1 report" |
| **Areas** | Ongoing responsibilities without end dates, standards to maintain | Health, Finance, Career Development |
| **Resources** | Topics of ongoing interest for reference | Note-taking, Programming, Cooking |
| **Archives** | Inactive items from other three categories | Completed projects, deprecated areas |

**The filing workflow (CODE framework):**
1. **C**apture to inbox
2. **O**rganize to PARA folders (check Projects first → Areas → Resources → else Archive)
3. **D**istill using progressive summarization
4. **E**xpress by creating outputs

**Best for:** Productivity-focused workers managing multiple active projects. When you need to reduce decision fatigue—determining actionability is faster than determining meaning.

**Core practices:**
- Keep hierarchies flat (maximum 2-3 levels deep)
- Move items between categories as actionability changes
- Put notes where they're actionable (e.g., "Tax Laws" inside "2024 Tax Return" project, not generic "Taxes" folder)

#### PARA Folder Structures

**Standard PARA hierarchy:**

```
📁 00 Inbox/                   ← Capture before organizing
📁 01 Projects/                ← Deadline-driven work
│   └── Project-Name/
│       └── README.md          ← Project index note
📁 02 Areas/                   ← Ongoing responsibilities
│   ├── Health/
│   ├── Finances/
│   ├── Career/
│   └── Relationships/
📁 03 Resources/               ← Reference materials by topic
│   ├── Note-taking/
│   ├── Programming/
│   └── Cooking/
📁 04 Archives/                ← Completed/inactive items
│   ├── Projects/
│   └── Areas/
📁 Templates/
```

**Tiago Forte's implementation principles:**
- **Maximum 4 levels deep** (Vault → PARA folder → Subfolder → Note)
- Only one subfolder under each PARA category
- Organize by action rather than topic

**Sébastien Dubois' nested archives pattern:**

```
📁 Areas/
│   ├── _Archives/             ← Area-specific archives
│   ├── Personal Development/
│   │   └── _Archives/         ← Nested archives
│   ├── Health/
│   └── Writing/
📁 Projects/
│   ├── _Archives/             ← Completed projects
│   └── Active Project/
```

Keeps archived material contextually close to active work while maintaining clean separation.

#### PARA Note Templates

**Project template:**
```markdown
---
note-type: project
status: active
area: [[]]
priority: medium
start-date: <% tp.date.now("YYYY-MM-DD") %>
due-date: 
tags: [project]
---

# <% tp.file.title %>

## Overview
_Quick overview, purpose, and success criteria_

## Goal
- [[Related Goal]]

## Success Criteria
- [ ] How do I know I'm done?

## Tasks
### Next Actions
- [ ] 

### Waiting For
- [ ] 

## Resources
- [[]]

## Journal
### <% tp.date.now("YYYY-MM-DD") %>
- 
```

#### PARA Essential Plugins

LifeOS (full PARA + Periodic Notes, won 3rd place in 2024 Obsidian Gems awards), QuickAdd (fast capture), Folder Notes (index notes for folders), Obsibrain (PARA-specific folder management)

---

### Maps of Content (MOC) / Linking Your Thinking

#### Overview

Nick Milo's Linking Your Thinking framework centers on **Maps of Content**—special notes that link to other notes, serving as navigational hubs. Unlike folders, a single note can appear in multiple MOCs, allowing flexible groupings and overlapping topics.

**Create an MOC when you experience "mental squeeze"**—typically when you have 6+ notes on a topic and feel overwhelmed finding things.

**Effective MOC structure:**
- Overview section
- Grouped links to related notes with brief context
- Questions still to explore
- Connections to related MOCs

**Best for:** Creative workers, those who dislike rigid folders, think non-linearly, and want emergent structure.

**Core practices:**
- Start MOCs organically rather than pre-planning them
- Use Dataview queries to auto-populate sections
- Delete MOCs that no longer serve you
- Notes can belong to multiple MOCs (unlike folders)

#### ACE Framework Structure

Nick Milo introduced the **ACE framework** (Atlas, Calendar, Efforts) in July 2023, mapping to the STIR model: **S**pace (Atlas), **T**ime (Calendar), **I**mportance (Efforts), **R**elatedness (Links). The Ideaverse for Obsidian has been downloaded over 55,000 times.

```
📁 + (Inbox)/                  ← Add/capture new notes
│   ├── Drafts/
│   ├── Notes/
│   └── Maps/

📁 Atlas/ (KNOWLEDGE)          ← Central hub for knowledge
│   ├── Ideas/                 ← Atomic idea notes
│   ├── Maps/                  ← MOCs for topics
│   │   └── My/                ← Personal maps (Compass, Values)
│   ├── Notes/
│   ├── Sources/
│   │   ├── Books/
│   │   ├── Articles/
│   │   └── Courses/
│   └── Utilities/
│       └── Templates/

📁 Calendar/ (TIME)            ← Time-based notes
│   ├── 2024/
│   │   └── 01-January/
│   ├── Journals/
│   └── Logs/

📁 Efforts/ (ACTION)           ← Replace rigid "Projects"
│   ├── 🔥 On/                 ← Active, high intensity
│   ├── ♻️ Ongoing/            ← Continuous efforts
│   ├── 〰️ Simmering/          ← Background, low priority
│   └── 💤 Sleeping/           ← Paused/archived

📁 x (Extras)/                 ← Support files
│   ├── Attachments/
│   └── CSS Snippets/
```

The "Efforts" concept replaces the rigidity of "Projects" (which must be completed) with intensity levels that allow work to ebb and flow naturally.

#### MOC Templates

**MOC template for topic navigation:**
```markdown
---
aliases: 
created: {{date:YYYY-MM-DD}}
tags: [moc]
up: [[Home]]
---

# {{title}}

> [!quote] Purpose
> This MOC gathers notes about [topic description].

## Core Concepts
- [[Concept A]]
- [[Concept B]]
- [[Concept C]]

## Related Ideas
- [[Related Idea 1]]
- [[Related Idea 2]]

## Questions to Explore
- What is...?
- How does...?

## Resources
- [[Source Note 1]]

---
**Related MOCs:** [[Parent MOC]] | [[Sibling MOC]]
```

**LYT-style home note:**
```markdown
# 🏠 Home

| I want to... | ...do this |
|---|---|
| 📥 Capture something | [[+ (Inbox)]] |
| 🧊 Process inbox | [[Cooling Pad]] |
| 🗃 Develop notes | [[Notebox]] |
| ✨ Work on ideas | [[Ideas Map]] |
| 🖌 Create output | [[Essays]] |

---

| Current Efforts | Status |
|---|---|
| [[Project Alpha]] | 🔥 Active |
| [[Learning Course]] | ♻️ Ongoing |
| [[Book Draft]] | 〰️ Simmering |

---

## 🧭 Compass
- [[Values]]
- [[Goals 2024]]
- [[Vision]]
```

---

### Johnny Decimal

#### Overview

Johnny Decimal uses a numbered taxonomy: **10-19** for broad categories, **11, 12, 13** for subcategories within each, and **11.01, 11.02** for individual items. Each item gets exactly one location.

**Best for:** File management with many non-note files (PDFs, images), team environments requiring shared structure, those who want predictable locations.

**Core constraints:**
- Maximum 10 top-level areas (00-09 through 90-99)
- Maximum 10 categories per area
- Unlimited items per category
- Each item lives in exactly one place

#### Johnny Decimal Folder Structure

```
📁 00-09 System/
│   ├── 00 Index/
│   │   ├── 00.00 Home.md
│   │   └── 00.01 Index.md
│   └── 01 Inbox/
│       └── 01.01 Quick Capture.md

📁 10-19 Personal/
│   ├── 11 Health/
│   │   ├── 11.01 Medical Records.md
│   │   └── 11.02 Fitness Log.md
│   ├── 12 Finance/
│   │   ├── 12.01 Budget.md
│   │   └── 12.02 Tax Documents/
│   └── 13 Home/

📁 20-29 Work/
│   ├── 21 Projects/
│   │   ├── 21.01 Project Alpha/
│   │   └── 21.02 Project Beta/
│   └── 22 Reference/

📁 30-39 Learning/
│   ├── 31 Courses/
│   └── 32 Books/
```

#### Johnny Decimal Index Note

```markdown
# Johnny Decimal Index

## 00-09 System
### 00 Index
- [[00.01 Master Index]]

### 01 Inbox  
- [[01.01 Quick Capture]]
- [[01.02 To Process]]

## 10-19 Personal
### 11 Me
- [[11.01 Health Records]]
- [[11.02 Goals]]

[...continue for all categories...]
```

#### Johnny Decimal Plugins

Folder Notes, Waypoint (auto-generates MOCs from folder trees), Another Quick Switcher (exact matching for JD numbers vs fuzzy matching)

**Practitioner caveat:** Large file counts (thousands of PDFs/receipts) can slow Obsidian plugins. Some practitioners maintain two parallel JD structures—one for the file system, one for notes.

---

### Hybrid Approaches

The most sophisticated vaults combine methodologies. **Critical insight: never nest Zettelkasten inside PARA—they should be parallel systems connected by links.**

#### Hub-and-Spoke: PARA + Zettelkasten

```
📁 00 Inbox/              ← Capture: fleeting notes, unprocessed ideas
📁 01 Projects/           ← PARA: Active work with deadlines
📁 02 Areas/              ← PARA: Ongoing responsibilities  
📁 03 Resources/          ← PARA: Literature notes, external content
📁 04 Archive/            ← PARA: Completed/inactive items
📁 10 Zettelkasten/       ← ZK: Permanent atomic notes (SEPARATE!)
📁 99 Templates/          ← System files
```

**Key principle: "Link, don't move."** Permanent notes stay in Zettelkasten but can be referenced from any project. When a project completes, archive the project folder—the permanent notes remain accessible.

#### Dusk's PARA + Zettelkasten Vault (379 GitHub stars)

```
📁 HUB/                  ← Central command
│   ├── Homepage.md
│   ├── Map of Content.md
│   └── Inbox.md
📁 PARA/                 ← Action-focused
│   ├── Projects/
│   ├── Areas/
│   ├── Resources/
│   └── Archive/
📁 ZETA/                 ← Idea-centric (Zettelkasten)
│   ├── Permanent Notes/
│   ├── Literature/
│   └── Fleeting Notes/
📁 DAILY/                ← Periodic notes
│   ├── Daily/
│   ├── Weekly/
│   └── Monthly/
📁 STICKY/               ← Temporary brainstorming
📁 SYSTEM/               ← Templates, media, configs
```

Features: homepage with quick navigation, Datacore-powered Map of Content, Focus Mode with Pomodoro. Available as both populated and empty vault downloads.

#### Kepano's Minimalist Bottom-Up Approach

Steph Ango (Obsidian CEO) maintains a public vault:

```
📁 Attachments/
📁 Categories/
📁 Clippings/
📁 Daily/
📁 Notes/
📁 References/
📁 Templates/
```

His style guide rules:
- Avoid splitting content into multiple vaults
- Avoid folders for organization
- Use internal links profusely
- Always pluralize categories and tags
- Use YYYY-MM-DD dates everywhere

#### How Folders, Links, and Tags Divide Labor

| Use Folders For | Use Links For | Use Tags For |
|-----------------|---------------|--------------|
| Note types (daily, project, reference) | Concept connections | Status (#toread, #processing) |
| Lifecycle stage (inbox → active → archive) | Cross-reference ideas | Note type (#quote, #question) |
| System files (templates, attachments) | Build thought chains | Quick filtering |

---

## Universal Note Templates

### Daily Note Template (Templater + Tasks)

```markdown
---
created: <% tp.file.creation_date() %>
tags: [daily]
week: <% tp.date.now("ww") %>
weekday: <% tp.date.now("dddd") %>
energy:
mood:
sleep:
---

# <% tp.file.title %>

<< [[<% tp.date.now("YYYY-MM-DD", -1, tp.file.title, "YYYY-MM-DD") %>]] | [[<% tp.date.now("YYYY-MM-DD", 1, tp.file.title, "YYYY-MM-DD") %>]] >>

## ✅ Tasks

### Due Today
```tasks
not done
due on <% tp.date.now("YYYY-MM-DD") %>
```

### New Tasks
- [ ] 

## 📝 Daily Log
- 

## 📓 Journal
### What went well today?
- 

### What could be improved?
- 
```

### Meeting Note Template

```markdown
---
date: <% tp.file.creation_date() %>
type: meeting
summary: " "
---

tags: [[🗣 Meetings MOC]]
Date: [[<% tp.date.now("YYYY-MM-DD-dddd") %>]]

# <% tp.file.title %>

**Attendees**: 
- [[]]

## Agenda/Questions
- 

## Notes
- 

## Action Items
- [ ] 

## Follow-up Required
- 
```

### Weekly Review Template

```markdown
---
note-type: weekly-review
week: <% tp.date.now("ww") %>
year: <% tp.date.now("YYYY") %>
tags: [review, weekly]
---

# Week <% tp.date.now("ww") %> - <% tp.date.now("YYYY") %>

<< [[<% tp.date.now("YYYY-[W]ww", -7) %>]] | [[<% tp.date.now("YYYY-[W]ww", 7) %>]] >>

## ✅ Completed This Week
```tasks
done after <% tp.date.now("YYYY-MM-DD", -7) %>
done before <% tp.date.now("YYYY-MM-DD", 1) %>
```

## 🎯 Review

### What went well this week?
- 

### What didn't go well?
- 

### What did I learn?
- 

## 📋 Next Week Planning

### Goals
1. 
2. 
3. 

### Key Tasks
- [ ] 
```

### Person/CRM Note Template

```markdown
---
tags: [person]
company: 
title:
email:
phone:
birthday:
met: 
last-contact: <% tp.date.now("YYYY-MM-DD") %>
---

# <% tp.file.title %>

## Notes
- 

## Meetings
```dataview
TABLE WITHOUT ID
  file.link as Meeting,
  file.cday as Date
FROM "Meetings"
WHERE contains(file.outlinks, [[<% tp.file.title %>]])
SORT file.cday DESC
```

## Interactions
- 
```

---

## Obsidian Configuration

### Essential Settings

**Editor settings:**
- Enable Live Preview mode
- Enable auto-pairing for brackets and Markdown syntax
- Enable readable line length

**Files & Links settings:**
- Set a specific inbox folder as default location for new notes
- Configure attachments to save in a dedicated folder
- **Critical:** Enable "Automatically update internal links" for safe refactoring
- Choose Wikilinks (Obsidian native, cleaner) or standard Markdown links (more portable)

### Daily Notes Organization

Organize by date hierarchy: `Daily Notes/2025/01-January/2025-01-24.md`

Set this pattern in Daily Notes settings using: `YYYY/MM-MMMM/YYYY-MM-DD`

This keeps thousands of daily notes organized without cluttering the root folder.

### File Naming Patterns

**Zettelkasten prefix style:** `202501241030-cognitive-load-theory.md`
- Provides unique IDs
- Chronological sorting
- Zero naming conflicts

**Declarative titles (Andy Matuschak style):** Phrase notes as claims
- "Working memory is limited to 4 chunks" instead of "Cognitive Load Theory"
- More linkable
- Forces clarity about what you actually believe

---

## Plugin Ecosystem

### Essential Plugin Stack (Most Users)

| Plugin | Purpose |
|--------|---------|
| **Calendar** | Visual navigation of daily notes |
| **Templater** | Advanced templates with logic and date manipulation |
| **Periodic Notes** | Weekly, monthly, quarterly note templates |
| **Tasks** | Task management with due dates, recurrence |

### Plugins by Workflow Type

**For Zettelkasten:**
- Dataview (query and display note metadata)
- Update Time on Edit
- Book Search
- Unique Note Creator

**For PARA:**
- LifeOS (full PARA + Periodic Notes integration)
- QuickAdd (fast capture)
- Folder Notes (index notes for folders)
- Obsibrain

**For MOCs/LYT:**
- Dataview (auto-populate MOC sections)
- Strange New Worlds (find unlinked mentions)

**For Johnny Decimal:**
- Folder Notes
- Waypoint (auto-generate MOCs from folders)
- Another Quick Switcher (exact matching)

### Integration Plugins

| Integration | Plugin/Method |
|-------------|---------------|
| **Readwise** | Official Readwise plugin (syncs highlights) |
| **Zotero** | Zotero Integration plugin (citations, annotations) |
| **Tasks** | Todoist Sync, Things 3 integration |
| **Calendar** | Full Calendar plugin, ICS import |

---

## Sustainable Practices

### Daily Practice Rhythm

| Time | Activity | Duration |
|------|----------|----------|
| **Morning** | Review day's tasks and calendar, pick 3 key outcomes | 10 min |
| **Throughout day** | Capture to inbox without processing | Ongoing |
| **Evening** | Quick processing of inbox, brief reflection | 10 min |

### Weekly Review (GTD-based)

**Get Clear:** Process all inboxes to zero, empty your head
**Get Current:** Review action lists, calendar, waiting items  
**Get Creative:** Review someday/maybe list, generate new ideas

Duration: 60-90 minutes. Schedule at fixed time (Friday afternoon, Sunday evening, Monday morning).

### Monthly Maintenance

- Archive finished projects
- Remove obsolete tags
- Turn note clusters into hub pages
- Write synthesis of what moved forward and what stalled

### The Five Pitfalls That Kill PKM Systems

| Pitfall | Description | Cure |
|---------|-------------|------|
| **Over-organization** | Building systems feels productive but is procrastination | Max 30 min daily on maintenance |
| **Perfectionism** | Prevents good-enough systems from working | Design for laziest version of yourself |
| **Collector's Fallacy** | Thinking collected = learned | Create more than you consume |
| **Shiny object syndrome** | Constant tool-switching | Commit to current tool for 6+ months |
| **Analysis paralysis** | Overthinking where to file things | Use simple structure, trust search |

### Progressive Summarization

Tiago Forte's layered approach for making notes discoverable:

| Layer | Action | Apply To |
|-------|--------|----------|
| **1** | Capture anything insightful | All notes |
| **2** | Bold the best parts | Most notes |
| **3** | Highlight the best of the bold | Some notes |
| **4** | Write executive summary | Few important notes |
| **5** | Remix into creative outputs | Tiny minority |

In Obsidian: Use **bold** for Layer 2 and ==highlights== for Layer 3.

**Critical:** Apply deeper layers only when already reviewing a note for another purpose—double-spend attention rather than creating separate "processing sessions."

---

## Downloadable Starter Vaults

| Vault | Focus | Source |
|-------|-------|--------|
| **Dusk PARA+ZK** | PARA + Zettelkasten hybrid | github.com/DuskWasHere/dusk-obsidian-vault |
| **Gröpl Zettelkasten Starter** | Pure Zettelkasten | github.com/groepl/Obsidian-Zettelkasten-Starter-Kit |
| **Ideaverse (LYT)** | MOC-focused, 300+ linked notes | linkingyourthinking.com/myideaverse/now |
| **Kepano** | Minimalist, bottom-up | github.com/kepano/kepano-obsidian |
| **byarbrough PARA** | Pure PARA method | github.com/byarbrough/obsidian-para |
| **LifeOS** | PARA + Periodic Notes plugin | github.com/quanru/obsidian-lifeos |
| **voidashi** | Second brain structure | github.com/voidashi/obsidian-vault-template |
| **Johnny Decimal Template** | JD + Obsidian index | github.com/DAWorm1/daw_johnny_decimal_obsidian |

---

## Method Selection Guide

| If you need... | Use this method |
|----------------|-----------------|
| Long-term knowledge compounding, research | Zettelkasten |
| Project management, reduce decision fatigue | PARA |
| Flexible navigation, creative work | Maps of Content |
| Strict file organization, team environments | Johnny Decimal |
| Both knowledge and project management | PARA + Zettelkasten hybrid |

**Key principle for all methods:** Begin with minimal folders organized by note type or workflow stage, not by topic. Let links create conceptual connections. Add structure only when you feel "mental squeeze points."
