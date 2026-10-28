---
type: moc
status: active
created: 2026-01-24
updated: '2026-10-03T09:10:00Z'
tags: [moc, home, dashboard]
cssclasses: [wide]
---
>
# Home

```datacorejsx
const { dailyPath } = await dc.require(dc.headerLink("_meta/dashboard-lib.md", "Daily note paths"));
const INBOX = "_inbox";
const FLEETING = "_templates/fleeting-note.md";
const pad = (x) => String(x).padStart(2, "0");
const CURSOR = /<%\s*tp\.file\.cursor\(\d*\)\s*%>/g;

// Open a note Templater just created and place the cursor at its tp.file.cursor() marker.
// Templater only does this itself when "Automatic jump to cursor" is on; any marker left over is removed.
async function openWithCursor(app, file) {
    if (!file) return;
    const leaf = app.workspace.getLeaf("tab");
    await leaf.openFile(file, { state: { mode: "source" } });
    app.workspace.setActiveLeaf(leaf, { focus: true });
    await app.plugins?.plugins?.["templater-obsidian"]?.editor_handler?.jump_to_next_cursor_location(file);
    await app.vault.process(file, (c) => c.replace(CURSOR, ""));
}

return function TodayHeader() {
    const now = new Date();
    const iso = `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())}`;
    const label = `${iso} · ${now.toLocaleDateString("en-GB", { weekday: "long" })}`;
    const path = dailyPath(iso);

    function openToday() {
        if (dc.app.vault.getAbstractFileByPath(path)) dc.app.workspace.openLinkText(path, "", "tab");
        else dc.app.commands.executeCommandById("periodic-notes:open-daily-note");
    }

    // Sections of this note, and the index note of each top-level folder.
    const PAGE = ["Habits", "Tasks", "Projects", "Areas", "Protocols"];
    const VAULT = [
        ["Inbox", "_inbox/_index.md"], ["Journal", "journal/_index.md"], ["Projects", "projects/_index.md"],
        ["Areas", "areas/_index.md"], ["Resources", "resources/_index.md"], ["Maps", "maps/_index.md"],
        ["Writing", "areas/writing/_index.md"], ["Meta", "_meta/_index.md"], ["Templates", "_templates/_index.md"],
    ];
    // Section jumps stay in this tab; notes open in a new one.
    const go = (target) => dc.app.workspace.openLinkText(target, "maps/home.md", target.startsWith("#") ? false : "tab");

    // Fleeting capture: a new note _inbox/<YYYY-MM-DD-HHmm>-<slug>.md from the fleeting template.
    // (The task bar under Tasks adds a task instead; it is unchanged.)
    const [idea, setIdea] = dc.useState("");
    async function captureFleeting() {
        const app = dc.app;
        const template = app.vault.getAbstractFileByPath(FLEETING);
        const folder = app.vault.getAbstractFileByPath(INBOX);
        const templater = app.plugins?.plugins?.["templater-obsidian"]?.templater;
        if (!template || !folder || !templater) {
            console.warn("Fleeting capture: template, _inbox/ or Templater not found");
            return;
        }
        const n = new Date();
        const stamp = `${n.getFullYear()}-${pad(n.getMonth() + 1)}-${pad(n.getDate())}-${pad(n.getHours())}${pad(n.getMinutes())}`;
        let slug = idea.trim().toLowerCase().replace(/[^a-z0-9äöüß]+/g, "-").replace(/^-+|-+$/g, "");
        if (slug.length > 60) slug = slug.slice(0, 61).replace(/-[^-]*$/, "") || slug.slice(0, 60);
        slug = slug || "capture";
        let name = `${stamp}-${slug}`;
        for (let i = 2; app.vault.getAbstractFileByPath(`${INBOX}/${name}.md`); i++) name = `${stamp}-${slug}-${i}`;
        await openWithCursor(app, await templater.create_new_note_from_template(template, folder, name, false));
        setIdea("");
    }

    return <div className="home-card">
        <div className="home-header">
            <div className="home-date">{label}</div>
            <button className="home-btn" onClick={openToday}>Open today's note</button>
        </div>
        <div className="home-nav">
            <span className="home-nav-label">Page</span>
            {PAGE.map(name => <span key={name} className="home-chip" onClick={() => go("#" + name)}>{name}</span>)}
        </div>
        <div className="home-nav">
            <span className="home-nav-label">Vault</span>
            {VAULT.map(([name, path]) => <span key={path} className="home-chip" onClick={() => go(path)}>{name}</span>)}
        </div>
        <div className="home-nav">
            <span className="home-nav-label">Capture</span>
            <input className="td-input" type="text" placeholder="Fleeting idea → _inbox/" value={idea}
                onInput={(e) => setIdea(e.currentTarget.value)}
                onKeyDown={(e) => { if (e.key === "Enter") captureFleeting(); }} />
            <button className="home-btn" onClick={captureFleeting}>New fleeting note</button>
        </div>
    </div>;
}
```

---

## Habits

```datacorejsx
// Habit tracker. Habits come from the registry in areas/self/habits-list.md;
// ticks are the #habit checkboxes in the daily notes, matched by [h::id].
const REGISTRY = "areas/self/habits-list.md";
const { DAILY, dailyPath, dayOf } = await dc.require(dc.headerLink("_meta/dashboard-lib.md", "Daily note paths"));
const DOW = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
const DAILY_TEMPLATE = "_templates/daily-note.md";

// A tick on a day without a daily note creates that note first, from the daily template via
// Templater, without opening it. One creation per day at a time (a double click waits for it).
const creating = new Map();
function ensureDailyNote(iso) {
    const app = dc.app;
    const path = dailyPath(iso);
    const existing = app.vault.getAbstractFileByPath(path);
    if (existing) return Promise.resolve(existing);
    if (!creating.has(iso)) {
        const template = app.vault.getAbstractFileByPath(DAILY_TEMPLATE);
        const templater = app.plugins?.plugins?.["templater-obsidian"]?.templater;
        if (!template || !templater) {
            console.warn("Habit tick: daily template or Templater not found");
            return Promise.resolve(null);
        }
        // Templater creates the year/month folder when it is missing.
        const folder = path.slice(0, path.lastIndexOf("/"));
        creating.set(iso, templater.create_new_note_from_template(template, folder, iso, false)
            .then((file) => file ?? null)
            .finally(() => creating.delete(iso)));
    }
    return creating.get(iso);
}

// ── Date helpers. Days are plain "YYYY-MM-DD" strings, so they compare with < and >. ──
const toISO = (v) => v == null ? null
    : v.toISODate ? v.toISODate()
    : v instanceof Date ? v.toISOString().slice(0, 10)
    : String(v).slice(0, 10);
const parse = (iso) => new Date(iso + "T00:00:00Z");
const fmt = (d) => d.toISOString().slice(0, 10);
const addDays = (iso, n) => { const d = parse(iso); d.setUTCDate(d.getUTCDate() + n); return fmt(d); };
const dowIndex = (iso) => (parse(iso).getUTCDay() + 6) % 7; // Mon = 0
const mondayOf = (iso) => addDays(iso, -dowIndex(iso));
const todayISO = () => {
    const n = new Date();
    const p = (x) => String(x).padStart(2, "0");
    return `${n.getFullYear()}-${p(n.getMonth() + 1)}-${p(n.getDate())}`;
};
const monthStart = (iso, offset) => {
    const d = parse(iso.slice(0, 8) + "01");
    d.setUTCMonth(d.getUTCMonth() + offset);
    return fmt(d);
};
const daysInMonth = (iso) => {
    const d = parse(iso);
    return new Date(Date.UTC(d.getUTCFullYear(), d.getUTCMonth() + 1, 0)).getUTCDate();
};
const MONTHS = ["January", "February", "March", "April", "May", "June", "July",
    "August", "September", "October", "November", "December"];

return function HabitDashboard() {
    const tasks = dc.useQuery(`@task and #habit and childof(@page and path("${DAILY}"))`);
    const regPages = dc.useQuery(`@page and path("${REGISTRY}")`);
    const [mode, setMode] = dc.useState("week");
    const [offset, setOffset] = dc.useState(0);
    const [showArchived, setShowArchived] = dc.useState(false);

    // Phone-sized screens get a tighter layout; desktop is unchanged. Re-checks on rotation.
    const [screenWidth, setScreenWidth] = dc.useState(window.innerWidth);
    dc.useEffect(() => {
        const onResize = () => setScreenWidth(window.innerWidth);
        window.addEventListener("resize", onResize);
        return () => window.removeEventListener("resize", onResize);
    }, []);
    const narrow = screenWidth < 600;
    const today = todayISO();

    // Days that have a daily note, so past days can be filled in from here.
    const dailyPages = dc.useQuery(`@page and path("${DAILY}")`);
    const noteDays = dc.useMemo(() => {
        const set = new Set();
        for (const p of dailyPages) {
            const day = dayOf(p.$path);
            if (day) set.add(day);
        }
        return set;
    }, [dailyPages]);

    // ── Registry: one entry per id, each with its list of dated rules. ──
    const registry = dc.useMemo(() => {
        const file = dc.app.vault.getAbstractFileByPath(REGISTRY);
        const raw = (file && dc.app.metadataCache.getFileCache(file)?.frontmatter?.habits) || [];
        const byId = new Map();
        for (const h of raw) {
            if (!h || !h.id) continue;
            const id = String(h.id);
            const rule = { ...h, id, since: toISO(h.since), until: toISO(h.until) };
            if (!byId.has(id)) byId.set(id, []);
            byId.get(id).push(rule);
        }
        const out = [];
        for (const [id, rules] of byId) {
            rules.sort((a, b) => (a.since ?? "") < (b.since ?? "") ? -1 : 1);
            const latest = rules[rules.length - 1];
            out.push({ id, name: latest.name ?? id, def: latest.def ?? "", block: latest.block ?? "",
                minutes: latest.minutes === true, until: latest.until,
                actions: Array.isArray(latest.actions) ? latest.actions : [], rules });
        }
        return out;
    }, [regPages]);

    // ── Log: id -> (day -> { status, task }), read from the daily notes. ──
    const log = dc.useMemo(() => {
        const m = new Map();
        for (const t of tasks) {
            const raw = t.value("h");
            if (raw == null) continue;
            const day = dayOf(t.$file);
            if (!day) continue;
            const id = String(raw).trim();
            if (!m.has(id)) m.set(id, new Map());
            const min = Number(t.value("min"));
            m.get(id).set(day, { status: t.$status, task: t, min: min > 0 ? min : 0 });
        }
        return m;
    }, [tasks]);

    function stateOf(id, day) {
        const e = log.get(id)?.get(day);
        if (!e) return null;
        if (e.status === "x" || e.status === "X") return "done";
        if (e.status === "-") return "skip";
        return "open";
    }

    function ruleFor(h, day) {
        let found = null;
        for (const r of h.rules) {
            if (r.since && r.since > day) continue;
            if (r.until && r.until < day) continue;
            found = r; // rules are sorted, so the last match is the newest
        }
        return found;
    }

    const perWeek = (r) => Number(r?.per_week) > 0 ? Number(r.per_week) : 0;

    const isOptional = (r) => r?.optional === true;

    // "Due" means a missed tick breaks the streak. Rest days and optional habits are simply not due.
    function isDue(r, day) {
        if (perWeek(r) || isOptional(r)) return false;
        if (Array.isArray(r.days) && r.days.length) return r.days.includes(DOW[dowIndex(day)]);
        return true;
    }

    function firstDay(h) {
        let first = h.rules[0].since;
        if (!first) {
            const days = [...(log.get(h.id)?.keys() ?? [])].sort();
            first = days[0] ?? today;
        }
        return first > today ? today : first;
    }

    // ── Streaks ──
    function stats(h) {
        const latest = ruleFor(h, today) ?? h.rules[h.rules.length - 1];
        const first = firstDay(h);
        let cur = 0, best = 0;

        // Optional habits have no streak; the period column shows their count or minutes.
        if (isOptional(latest)) return { cur: null, best: null, unit: "", progress: null };

        if (!perWeek(latest)) {
            for (let d = first; d <= today; d = addDays(d, 1)) {
                const r = ruleFor(h, d);
                if (!r) continue;
                const s = stateOf(h.id, d);
                if (s === "done") { cur++; if (cur > best) best = cur; continue; }
                if (s === "skip") continue;       // planned skip: neutral
                if (!isDue(r, d)) continue;       // rest day: neutral
                if (d === today) continue;        // today is not over yet
                cur = 0;                          // due and missed
            }
            return { cur, best, unit: "d", progress: null };
        }

        // N-times-a-week habits: the streak counts weeks that hit the target.
        const thisMonday = mondayOf(today);
        let progress = null;
        for (let w = mondayOf(first); w <= today; w = addDays(w, 7)) {
            let done = 0, skips = 0;
            for (let i = 0; i < 7; i++) {
                const s = stateOf(h.id, addDays(w, i));
                if (s === "done") done++;
                else if (s === "skip") skips++;
            }
            const weekEnd = addDays(w, 6);
            const r = ruleFor(h, weekEnd > today ? today : weekEnd) ?? latest;
            const target = Math.max(0, (perWeek(r) || perWeek(latest)) - skips);
            const isCurrent = w === thisMonday;
            if (isCurrent) progress = `${done}/${target}`;
            if (target === 0 && done === 0) continue;   // whole week skipped
            if (done >= target) { cur++; if (cur > best) best = cur; }
            else if (!isCurrent) cur = 0;
        }
        return { cur, best, unit: "wk", progress };
    }

    // ── Visible period ──
    let days = [], label = "";
    if (mode === "week") {
        const start = addDays(mondayOf(today), offset * 7);
        for (let i = 0; i < 7; i++) days.push(addDays(start, i));
        label = `${start} – ${addDays(start, 6)}`;
    } else {
        const start = monthStart(today, offset);
        const n = daysInMonth(start);
        for (let i = 0; i < n; i++) days.push(addDays(start, i));
        label = `${MONTHS[Number(start.slice(5, 7)) - 1]} ${start.slice(0, 4)}`;
    }

    function cell(h, d) {
        const s = stateOf(h.id, d);
        if (s === "done") return { sym: "✓", color: "var(--color-green)", title: "Done" };
        if (s === "skip") return { sym: "–", color: "var(--text-faint)", title: "Planned skip" };
        const r = ruleFor(h, d);
        if (!r || d > today) return { sym: "", color: "var(--text-faint)", title: "" };
        if (!isDue(r, d)) return { sym: "·", color: "var(--text-faint)",
            title: isOptional(r) ? "Optional" : perWeek(r) ? "Not done" : "Rest day" };
        if (d === today) return { sym: "○", color: "var(--text-muted)", title: "Open" };
        return { sym: "✗", color: "var(--color-red)", title: "Missed" };
    }

    function periodSummary(h) {
        let done = 0, due = 0, min = 0;
        for (const d of days) {
            if (d > today) break;
            const r = ruleFor(h, d);
            if (!r) continue;
            const s = stateOf(h.id, d);
            if (s === "done") { done++; due++; min += log.get(h.id)?.get(d)?.min ?? 0; }
            else if (s !== "skip" && isDue(r, d)) due++;
        }
        const latest = ruleFor(h, today) ?? h.rules[h.rules.length - 1];
        if (h.minutes && min > 0) return `${min}m`;
        if (perWeek(latest) || isOptional(latest)) return `${done}`;
        return due ? `${Math.round((100 * done) / due)}%` : "–";
    }

    // ── Writing a tick back into the daily note ──
    async function setStatus(h, d, next) {
        const entry = log.get(h.id)?.get(d);
        let file = dc.app.vault.getAbstractFileByPath(entry ? entry.task.$file : dailyPath(d));
        // No note for that day: a tick or skip creates it; clearing a mark never does.
        if (!file && next !== " ") file = await ensureDailyNote(d);
        if (!file) return;
        const marker = new RegExp(`\\[h::\\s*${h.id}\\s*\\]`);
        const isBox = (l) => /^\s*[-*+] \[.\]/.test(l) && marker.test(l);
        // `logged` records when the tracker was last filled from here (local time), so a fill
        // on a later day shows up as retroactive. Only the `logged:` line is touched.
        const n = new Date();
        const p = (x) => String(x).padStart(2, "0");
        const loggedAt = `${todayISO()}T${p(n.getHours())}:${p(n.getMinutes())}`;
        function stampLogged(lines) {
            if (lines[0] !== "---") return;
            const end = lines.indexOf("---", 1);
            if (end < 0) return;
            const k = lines.slice(1, end).findIndex(l => /^logged:/.test(l));
            if (k >= 0) lines[k + 1] = `logged: ${loggedAt}`;
            else lines.splice(end, 0, `logged: ${loggedAt}`);
        }
        await dc.app.vault.process(file, (content) => {
            const lines = content.split("\n");
            let i = entry ? (entry.task.$position?.start ?? -1) : -1;
            if (i < 0 || i >= lines.length || !isBox(lines[i])) i = lines.findIndex(isBox);
            if (i >= 0) {
                lines[i] = lines[i].replace(/^(\s*[-*+] \[).(\])/, `$1${next}$2`);
                stampLogged(lines);
                return lines.join("\n");
            }
            stampLogged(lines);
            // The note has no line for this habit yet: add one after the last habit line.
            const line = `- [${next}] ${h.name} #habit [h::${h.id}]${h.minutes ? " [min:: ]" : ""}`;
            let last = -1;
            lines.forEach((l, idx) => { if (/^\s*[-*+] \[.\].*#habit/.test(l)) last = idx; });
            if (last < 0) last = lines.findIndex(l => /^#{1,6}\s.*Habits\s*$/.test(l));
            if (last < 0) return content.replace(/\s*$/, "") + `\n\n## ✅ Habits\n${line}\n`;
            lines.splice(last + 1, 0, line);
            return lines.join("\n");
        });
    }

    // A cell can be clicked today or earlier when that day has a daily note, or when the habit is
    // in the registry for that day (the click then creates the note; see setStatus).
    const canClick = (h, d) => d <= today && (stateOf(h.id, d) != null || noteDays.has(d) || ruleFor(h, d) != null);

    // Habit buttons: create a note from a template in the right folder, or open a note.
    async function runAction(action) {
        const app = dc.app;
        if (action.open) {
            const isToday = action.open === "today";
            const path = isToday ? dailyPath(today) : String(action.open);
            if (app.vault.getAbstractFileByPath(path)) app.workspace.openLinkText(path, "", "tab");
            else if (isToday) app.commands.executeCommandById("periodic-notes:open-daily-note");
            return;
        }
        const template = app.vault.getAbstractFileByPath(String(action.template ?? ""));
        const folder = app.vault.getAbstractFileByPath(String(action.folder ?? ""));
        const templater = app.plugins?.plugins?.["templater-obsidian"]?.templater;
        if (!template || !folder || !templater) {
            console.warn("Habit action: template, folder or Templater not found", action);
            return;
        }
        const file = await templater.create_new_note_from_template(template, folder, undefined, false);
        if (!file) return;
        // Open in a new tab and place the cursor at tp.file.cursor() (Templater skips this unless
        // "Automatic jump to cursor" is on); any marker left over is removed.
        const leaf = app.workspace.getLeaf("tab");
        await leaf.openFile(file, { state: { mode: "source" } });
        app.workspace.setActiveLeaf(leaf, { focus: true });
        await app.plugins?.plugins?.["templater-obsidian"]?.editor_handler?.jump_to_next_cursor_location(file);
        await app.vault.process(file, (c) => c.replace(/<%\s*tp\.file\.cursor\(\d*\)\s*%>/g, ""));
    }

    function onCellClick(e, h, d) {
        if (!canClick(h, d)) return; // future day, or no note and the habit not active then
        const s = stateOf(h.id, d);
        const next = e.shiftKey ? (s === "skip" ? " " : "-") : (s === "done" ? " " : "x");
        setStatus(h, d, next);
    }

    // ── Rendering ──
    const compact = mode === "month";
    const border = "1px solid var(--background-modifier-border)";
    // The grid is built from plain boxes (CSS grid), not an HTML table, so theme table styles don't move it.
    const box = { display: "flex", alignItems: "center", justifyContent: "center", borderBottom: border,
        padding: compact ? "2px 3px" : (narrow ? "7px 4px" : "5px 6px"), minWidth: 0,
        ...(narrow ? { fontSize: "0.9em" } : {}) };
    const th = { ...box, fontWeight: 500, fontSize: compact ? "0.7em" : "0.8em",
        color: "var(--text-muted)", whiteSpace: "nowrap" };
    const td = { ...box };
    // On a phone the habit names stay pinned while the days scroll underneath.
    const pinned = narrow ? { position: "sticky", left: 0, zIndex: 1, background: "var(--background-secondary)" } : {};
    const left = { justifyContent: "flex-start", textAlign: "left", flexWrap: "wrap", ...pinned,
        ...(narrow ? { flexDirection: "column", alignItems: "flex-start", justifyContent: "center" } : {}) };
    // Phone portrait keeps full labels and all columns; the grid scrolls sideways past the pinned names.
    const nameCol = narrow ? 110 : (compact ? 120 : 260);
    const dayCol = compact ? 20 : 56;
    const tailCols = "repeat(3, minmax(52px, auto))";
    const cols = `minmax(${nameCol}px, ${narrow ? "1.4fr" : "3fr"}) repeat(${days.length}, minmax(${dayCol}px, 1fr)) ${tailCols}`;
    const btn = { padding: "2px 10px", cursor: "pointer" };
    const todayBg = "var(--background-modifier-hover)";

    // Archived = retired with an `until:` in the past. Everything else is active, including
    // habits whose `since` is still ahead (their cells stay empty until then).
    const isArchived = (entry) => entry.until != null && entry.until < today;
    const activeRows = registry.filter(entry => !isArchived(entry));
    const archivedRows = registry.filter(isArchived);
    const rows = showArchived ? archivedRows : activeRows;

    if (!registry.length) {
        return <p>No habits found. Add a <code>habits:</code> list to the properties of <code>{REGISTRY}</code>.</p>;
    }

    return <div className="home-card">
        <div style={{ display: "flex", gap: "6px", alignItems: "center", marginBottom: "8px", flexWrap: "wrap" }}>
            <button style={btn} onClick={() => { setMode("week"); setOffset(0); }} disabled={mode === "week"}>Week</button>
            <button style={btn} onClick={() => { setMode("month"); setOffset(0); }} disabled={mode === "month"}>Month</button>
            <span className="home-chip" title="Edit habits, definitions and order"
                onClick={() => dc.app.workspace.openLinkText(REGISTRY, "", "tab")}>Habit list</span>
            {archivedRows.length > 0 && <span className="home-chip" title="Retired habits and their history"
                onClick={() => setShowArchived(!showArchived)}>
                {showArchived ? "Back to active" : `Archived (${archivedRows.length})`}</span>}
            <span style={{ flex: 1 }}></span>
            <button style={btn} onClick={() => setOffset(offset - 1)}>←</button>
            <span style={{ minWidth: "11em", textAlign: "center", fontWeight: 600 }}>{label}</span>
            <button style={btn} onClick={() => setOffset(offset + 1)} disabled={offset >= 0}>→</button>
            <button style={btn} onClick={() => setOffset(0)} disabled={offset === 0}>Today</button>
        </div>
        <div style={{ overflowX: "auto" }}>
            <div style={{ display: "grid", gridTemplateColumns: cols, minWidth: compact ? "820px" : "0" }}>
                <div style={{ ...th, ...left }}>Habit</div>
                {days.map(d => <div key={"head:" + d} style={{ ...th, background: d === today ? todayBg : "transparent" }}>
                    {compact ? Number(d.slice(8)) : `${DOW[dowIndex(d)]} ${Number(d.slice(8))}`}
                </div>)}
                <div style={th}>{compact ? "Month" : "Week"}</div>
                <div style={th}>Streak</div>
                <div style={th}>Best</div>
                {rows.map((habit, idx) => {
                    const s = stats(habit);
                    // A heading row wherever the routine block changes (active view only).
                    const blockRow = !showArchived && habit.block && habit.block !== rows[idx - 1]?.block
                        ? <div key={habit.id + ":block"} style={{ ...th, ...left, gridColumn: "1 / -1",
                            flexDirection: "row", justifyContent: "flex-start", paddingTop: idx ? "10px" : undefined,
                            color: "var(--text-normal)", fontWeight: 600 }}>{habit.block}</div>
                        : null;
                    return [
                        blockRow,
                        <div key={habit.id + ":name"} title={habit.def || undefined}
                            style={{ ...td, ...left, whiteSpace: compact ? "nowrap" : "normal",
                                cursor: habit.def ? "help" : "default" }}>
                            {habit.name}
                            {habit.actions.map(action => <span key={action.label} title={action.label}
                                style={{ marginLeft: narrow ? 0 : "8px", fontSize: "0.75em", cursor: "pointer",
                                    color: "var(--text-accent)", whiteSpace: "nowrap" }}
                                onClick={() => runAction(action)}>+ {action.label}</span>)}
                        </div>,
                        ...days.map(d => {
                            const c = cell(habit, d);
                            const clickable = canClick(habit, d);
                            return <div key={habit.id + ":" + d} title={c.title}
                                style={{ ...td, color: c.color, fontWeight: 600,
                                    cursor: clickable ? "pointer" : "default",
                                    background: d === today ? todayBg : "transparent" }}
                                onClick={(e) => onCellClick(e, habit, d)}>{c.sym}</div>;
                        }),
                        <div key={habit.id + ":period"} style={{ ...td, color: "var(--text-muted)" }}>
                            {!compact && s.progress && offset === 0 ? s.progress : periodSummary(habit)}
                        </div>,
                        <div key={habit.id + ":streak"} style={{ ...td, fontWeight: 600, whiteSpace: "nowrap" }}>{s.cur == null ? "–" : `${s.cur} ${s.unit}`}</div>,
                        <div key={habit.id + ":best"} style={{ ...td, color: "var(--text-muted)", whiteSpace: "nowrap" }}>{s.best == null ? "–" : `${s.best} ${s.unit}`}</div>,
                    ];
                })}
            </div>
        </div>
        <div style={{ fontSize: "0.75em", color: "var(--text-faint)", marginTop: "6px" }}>
            ✓ done · ✗ missed · – planned skip · · rest day / not done / optional · ○ open today
        </div>
    </div>;
}
```

---

## Tasks

```datacorejsx
// Todoist-style task view. Reads every checkbox under journal/ that is not a #habit.
// New tasks go into the daily note of their due date (today's note when there is no due date)
// under "## Tasks"; when that daily note does not exist, they go into journal/tasks.md.
const SCOPE = "journal";
const { dailyPath, dayOf } = await dc.require(dc.headerLink("_meta/dashboard-lib.md", "Daily note paths"));
const FALLBACK = "journal/tasks.md";
const PRIO_MARK = { 1: "⏫", 2: "🔼", 3: "🔽" };
const WEEKDAYS = ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"];
const MONTHS = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
const ISO = /^\d{4}-\d{2}-\d{2}$/;

const pad = (x) => String(x).padStart(2, "0");
const isoOf = (d) => `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;
const todayISO = () => isoOf(new Date());
const addDays = (iso, n) => { const d = new Date(iso + "T12:00:00"); d.setDate(d.getDate() + n); return isoOf(d); };
const toISO = (v) => v == null ? null
    : v.toISODate ? v.toISODate()
    : v instanceof Date ? isoOf(v)
    : String(v).slice(0, 10);

// All dates are shown as YYYY-MM-DD, matching the daily note names.
const dayLabel = (due) => `${due} · ${WEEKDAYS[new Date(due + "T12:00:00").getDay()]}`;

// Date box: accepts YYYY-MM-DD, or the shortcuts today / tomorrow / +3 / mon..sun.
// Returns "" for no date and null for something it cannot read.
function resolveDate(input) {
    const s = String(input ?? "").trim().toLowerCase();
    if (!s) return "";
    const today = todayISO();
    if (ISO.test(s)) return isNaN(new Date(s + "T12:00:00").getTime()) ? null : s;
    if (s === "today" || s === "tod") return today;
    if (s === "tomorrow" || s === "tom") return addDays(today, 1);
    const plus = s.match(/^\+(\d{1,3})$/);
    if (plus) return addDays(today, Number(plus[1]));
    if (s.length >= 3) {
        const wd = WEEKDAYS.findIndex(w => w.toLowerCase().startsWith(s));
        if (wd >= 0) {
            let diff = (wd - new Date().getDay() + 7) % 7;
            if (diff === 0) diff = 7;
            return addDays(today, diff);
        }
    }
    return null;
}

// Turn a Datacore task into a plain object the view can use.
function parseTask(t) {
    const raw = String(t.$text ?? "").split("\n")[0];
    let due = null;
    const dueMatch = raw.match(/📅\s*(\d{4}-\d{2}-\d{2})/u);
    if (dueMatch) due = dueMatch[1];
    else { const v = toISO(t.value("due")); if (v && ISO.test(v)) due = v; }

    let prio = 4;
    if (raw.includes("🔺") || raw.includes("⏫")) prio = 1;
    else if (raw.includes("🔼")) prio = 2;
    else if (raw.includes("🔽") || raw.includes("⏬")) prio = 3;

    const doneMatch = raw.match(/✅\s*(\d{4}-\d{2}-\d{2})/u);
    const noteDay = dayOf(t.$file); // only a daily note dates its tasks; reviews and tasks.md do not
    const text = raw
        .replace(/(📅|✅|➕|🛫|⏳)\s*\d{4}-\d{2}-\d{2}/gu, "")
        .replace(/\[(due|completion|done)::[^\]]*\]/gi, "")
        .replace(/(🔺|⏫|🔼|🔽|⏬)/gu, "")
        .replace(/\s+/g, " ")
        .trim();

    return {
        key: `${t.$file}:${t.$position?.start}`,
        task: t, raw, text, due, prio,
        file: t.$file ?? "",
        status: t.$status,
        done: t.$completed === true,
        doneDate: doneMatch ? doneMatch[1] : noteDay,
    };
}

const bodyOf = (text, due, prio) =>
    [text.trim(), PRIO_MARK[prio] ?? "", due ? `📅 ${due}` : ""].filter(Boolean).join(" ");

const TASK_LINE = /^(\s*[-*+] \[)(.)(\]\s?)(.*)$/;

// Find the task's line in its file and replace it (or delete it when fn returns null).
async function editLine(item, fn) {
    const file = dc.app.vault.getAbstractFileByPath(item.file);
    if (!file) return;
    const matches = (l) => TASK_LINE.test(l) && (!item.raw || l.includes(item.raw));
    await dc.app.vault.process(file, (content) => {
        const lines = content.split("\n");
        let i = item.task.$position?.start ?? -1;
        if (!(i >= 0 && i < lines.length && matches(lines[i]))) i = lines.findIndex(matches);
        if (i < 0) return content;
        const next = fn(lines[i]);
        if (next == null) lines.splice(i, 1);
        else lines[i] = next;
        return lines.join("\n");
    });
}

async function toggleTask(item) {
    const today = todayISO();
    await editLine(item, (line) => {
        const m = line.match(TASK_LINE);
        if (!m) return line;
        if (m[2] === " ") return `${m[1]}x${m[3]}${m[4].trimEnd()} ✅ ${today}`;
        return `${m[1]} ${m[3]}${m[4].replace(/\s*✅\s*\d{4}-\d{2}-\d{2}/u, "")}`;
    });
}

async function saveTask(item, text, due, prio) {
    await editLine(item, (line) => {
        const m = line.match(TASK_LINE);
        if (!m) return line;
        const done = m[4].match(/✅\s*\d{4}-\d{2}-\d{2}/u);
        return `${m[1]}${m[2]}${m[3]}${bodyOf(text, due, prio)}${done ? " " + done[0] : ""}`;
    });
}

async function deleteTask(item) {
    await editLine(item, () => null);
}

async function addTask(text, due, prio) {
    const today = todayISO();
    const line = `- [ ] ${bodyOf(text, due, prio)}`;
    const vault = dc.app.vault;
    const daily = vault.getAbstractFileByPath(dailyPath(due || today));

    if (daily) {
        await vault.process(daily, (content) => {
            const lines = content.split("\n");
            const head = lines.findIndex(l => /^#{1,6}\s+(\S+\s+)?Tasks\s*$/.test(l));
            if (head < 0) return content.replace(/\s*$/, "") + `\n\n## Tasks\n${line}\n`;
            let end = lines.length;
            for (let i = head + 1; i < lines.length; i++) {
                if (/^#{1,6}\s/.test(lines[i])) { end = i; break; }
            }
            // Insert after the last task, above trailing blank lines and a closing --- rule.
            let at = end;
            while (at - 1 > head && /^(\s*|-{3,}|\*{3,}|_{3,})$/.test(lines[at - 1].trim())) at--;
            lines.splice(at, 0, line);
            return lines.join("\n");
        });
        return;
    }

    const inbox = vault.getAbstractFileByPath(FALLBACK);
    if (inbox) {
        await vault.process(inbox, (content) => content.replace(/\s*$/, "") + `\n${line}\n`);
    } else {
        await vault.create(FALLBACK,
            `---\ntype: journal\nstatus: active\ncreated: ${today}\ntags: [tasks]\n---\n\n# Task Inbox\n\n` +
            `Tasks added from the home dashboard on days without a daily note.\n\n${line}\n`);
    }
}

function PrioSelect({ value, onChange }) {
    return <select value={String(value)} onChange={(e) => onChange(Number(e.currentTarget.value))}>
        <option value="1">P1 urgent</option>
        <option value="2">P2 high</option>
        <option value="3">P3 medium</option>
        <option value="4">P4 none</option>
    </select>;
}

function AddBar() {
    const [text, setText] = dc.useState("");
    const [due, setDue] = dc.useState("");
    const [prio, setPrio] = dc.useState(4);

    async function submit() {
        const date = resolveDate(due);
        if (!text.trim() || date === null) return;
        await addTask(text, date, prio);
        setText(""); setDue(""); setPrio(4);
    }

    return <div className="td-add">
        <input className="td-input" type="text" placeholder="Add a task…" value={text}
            onInput={(e) => setText(e.currentTarget.value)}
            onKeyDown={(e) => { if (e.key === "Enter") submit(); }} />
        <input className={"td-date" + (resolveDate(due) === null ? " is-bad" : "")} type="text"
            placeholder="YYYY-MM-DD" title="YYYY-MM-DD, today, tomorrow, +3, mon…sun" value={due}
            onInput={(e) => setDue(e.currentTarget.value)}
            onKeyDown={(e) => { if (e.key === "Enter") submit(); }} />
        <PrioSelect value={prio} onChange={setPrio} />
        <button className="td-btn td-btn-primary" onClick={submit}>Add</button>
    </div>;
}

function Row({ item, today }) {
    const [editing, setEditing] = dc.useState(false);
    const [text, setText] = dc.useState(item.text);
    const [due, setDue] = dc.useState(item.due ?? "");
    const [prio, setPrio] = dc.useState(item.prio);
    const [confirm, setConfirm] = dc.useState(false);

    function startEdit() {
        setText(item.text); setDue(item.due ?? ""); setPrio(item.prio);
        setConfirm(false); setEditing(true);
    }
    async function save() {
        const date = resolveDate(due);
        if (!text.trim() || date === null) return;
        await saveTask(item, text, date, prio);
        setEditing(false);
    }
    async function remove() {
        if (!confirm) { setConfirm(true); return; }
        await deleteTask(item);
        setEditing(false);
    }

    if (editing) {
        return <div className="td-edit">
            <input className="td-input" type="text" value={text}
                onInput={(e) => setText(e.currentTarget.value)}
                onKeyDown={(e) => {
                    if (e.key === "Enter") save();
                    if (e.key === "Escape") setEditing(false);
                }} />
            <input className={"td-date" + (resolveDate(due) === null ? " is-bad" : "")} type="text"
                placeholder="YYYY-MM-DD" title="YYYY-MM-DD, today, tomorrow, +3, mon…sun" value={due}
                onInput={(e) => setDue(e.currentTarget.value)}
                onKeyDown={(e) => { if (e.key === "Enter") save(); if (e.key === "Escape") setEditing(false); }} />
            <PrioSelect value={prio} onChange={setPrio} />
            <button className="td-btn td-btn-primary" onClick={save}>Save</button>
            <button className="td-btn" onClick={() => setEditing(false)}>Cancel</button>
            <button className="td-btn td-btn-danger" onClick={remove}>{confirm ? "Really delete?" : "Delete"}</button>
        </div>;
    }

    const fileName = item.file.split("/").pop().replace(/\.md$/, "");
    const dueClass = item.due < today ? " is-overdue" : item.due === today ? " is-today" : "";
    return <div className={"td-row" + (item.done ? " is-done" : "")}>
        <span className={`td-check td-p${item.prio}` + (item.done ? " is-done" : "")}
            title={item.done ? "Mark as open" : "Complete"}
            onClick={() => toggleTask(item)}></span>
        <span className="td-text" title="Click to edit" onClick={startEdit}>{item.text}</span>
        {item.due && !item.done && <span className={"td-due" + dueClass}>{item.due}</span>}
        <span className="td-src" title="Open note"
            onClick={() => dc.app.workspace.openLinkText(item.file, "", "tab")}>{fileName}</span>
    </div>;
}

function Rows({ list, today, empty }) {
    if (!list.length) return empty ? <div className="td-empty">{empty}</div> : null;
    return <div>{list.map(item => <Row key={item.key} item={item} today={today} />)}</div>;
}

return function TaskDashboard() {
    const tasks = dc.useQuery(`@task and !#habit and childof(@page and path("${SCOPE}"))`);
    const [tab, setTab] = dc.useState("today");
    const today = todayISO();

    const items = dc.useMemo(() => tasks.map(parseTask).filter(i => i.text), [tasks]);
    const open = items.filter(i => !i.done && i.status !== "-");
    const byDueThenPrio = (a, b) => a.due.localeCompare(b.due) || a.prio - b.prio;

    const overdue = open.filter(i => i.due && i.due < today).sort(byDueThenPrio);
    const dueToday = open.filter(i => i.due === today).sort((a, b) => a.prio - b.prio);
    const upcoming = open.filter(i => i.due && i.due > today).sort(byDueThenPrio);
    const inbox = open.filter(i => !i.due).sort((a, b) => a.prio - b.prio || b.file.localeCompare(a.file));
    const cutoff = addDays(today, -14);
    const done = items.filter(i => i.done && i.doneDate && i.doneDate >= cutoff)
        .sort((a, b) => b.doneDate.localeCompare(a.doneDate));

    // Upcoming, grouped by day.
    const upcomingDays = [];
    for (const item of upcoming) {
        const last = upcomingDays[upcomingDays.length - 1];
        if (last && last.due === item.due) last.list.push(item);
        else upcomingDays.push({ due: item.due, list: [item] });
    }

    const tabs = [
        ["today", "Today", overdue.length + dueToday.length],
        ["upcoming", "Upcoming", upcoming.length],
        ["inbox", "No date", inbox.length],
        ["done", "Done", done.length],
    ];

    return <div className="td">
        <AddBar />
        <div className="td-tabs">
            {tabs.map(([id, label, count]) =>
                <span key={id} className={"td-tab" + (tab === id ? " is-active" : "")} onClick={() => setTab(id)}>
                    {label}<span className="td-count">{count}</span>
                </span>)}
        </div>

        {tab === "today" && <div>
            {overdue.length > 0 && <div className="td-section-title is-overdue">Overdue</div>}
            <Rows list={overdue} today={today} />
            {overdue.length > 0 && dueToday.length > 0 && <div className="td-section-title">Today</div>}
            <Rows list={dueToday} today={today} />
            {overdue.length + dueToday.length === 0 &&
                <div className="td-empty">Nothing due today. Undated tasks are under "No date".</div>}
        </div>}

        {tab === "upcoming" && <div>
            {upcomingDays.map(group => <div key={group.due}>
                <div className="td-section-title">{dayLabel(group.due)}</div>
                <Rows list={group.list} today={today} />
            </div>)}
            {upcoming.length === 0 && <div className="td-empty">No upcoming tasks.</div>}
        </div>}

        {tab === "inbox" && <Rows list={inbox} today={today} empty="No undated tasks." />}
        {tab === "done" && <Rows list={done} today={today} empty="Nothing completed in the last 14 days." />}
    </div>;
}
```

---

## Projects

```datacorejsx
// Notes under projects/ with `type: project` in their properties.
const ROOT = "projects";
const pretty = (slug) => String(slug).split("-").map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(" ");
const toISO = (v) => v == null || v === "" ? null
    : v.toISODate ? v.toISODate()
    : String(v).slice(0, 10);

return function Projects() {
    const pages = dc.useQuery(`@page and path("${ROOT}")`);
    const [tab, setTab] = dc.useState("active");

    const items = dc.useMemo(() => pages
        .filter(p => p.value("type") === "project")
        .map(p => ({
            path: p.$path,
            name: pretty(String(p.$path).split("/").pop().replace(/\.md$/, "")),
            status: String(p.value("status") ?? "unset"),
            due: toISO(p.value("due")),
        }))
        .sort((a, b) => a.name.localeCompare(b.name)), [pages]);

    const active = items.filter(i => i.status === "active");
    const other = items.filter(i => i.status !== "active");
    const list = tab === "active" ? active : other;
    const go = (path) => dc.app.workspace.openLinkText(path, "", "tab");

    return <div className="td">
        <div className="td-tabs">
            {[["active", "Active", active.length], ["other", "Paused / done", other.length]].map(([id, label, count]) =>
                <span key={id} className={"td-tab" + (tab === id ? " is-active" : "")} onClick={() => setTab(id)}>
                    {label}<span className="td-count">{count}</span>
                </span>)}
        </div>
        <div>
            {list.map(item => <div key={item.path} className="td-row td-wrap">
                <span className={"td-dot" + (item.status === "active" ? " is-active" : " is-paused")}></span>
                <span className="td-text home-link" onClick={() => go(item.path)}>{item.name}</span>
                {item.due && <span className="td-due">{item.due}</span>}
                <span className="td-chip">{item.status}</span>
            </div>)}
            {list.length === 0 && <div className="td-empty">Nothing here.</div>}
        </div>
    </div>;
}
```

---

## Areas

```datacorejsx
// Built from the notes that actually exist under areas/, so the links cannot go stale.
const ROOT = "areas";
const DESCRIPTIONS = {
    "admin": "Cases (Behörden, Gläubiger, Gericht), correspondence, accounts",
    "finance": "Income, recurring, debts, budget",
    "health": "Fasting, exercise, sleep, biohacking",
    "career": "Professional development",
    "self": "Typology, goals, philosophy, habits, learning",
    "protocols": "Supplements, compounds, skincare, abstinence, diets",
    "writing": "Blog drafts and published posts",
    "infrastructure": "Machines: arch-thinkpad, netcup-vps, pixel-graphene, shared",
    "freelance": "Freelance practice: clients, standards, business",
};
const titleOf = (slug) => slug.split("-").map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(" ");

return function Areas() {
    const pages = dc.useQuery(`@page and path("${ROOT}")`);
    const [openArea, setOpenArea] = dc.useState(null);

    const areas = dc.useMemo(() => {
        const map = new Map();
        for (const p of pages) {
            const parts = String(p.$path).split("/");
            if (parts.length < 3) continue; // skip areas/_index.md
            const slug = parts[1];
            if (!map.has(slug)) map.set(slug, { slug, index: null, notes: [] });
            const rest = parts.slice(2).join("/").replace(/\.md$/, "");
            if (rest === "_index") map.get(slug).index = p.$path;
            else map.get(slug).notes.push({ path: p.$path, label: rest });
        }
        const out = [...map.values()];
        out.sort((a, b) => a.slug.localeCompare(b.slug));
        for (const a of out) a.notes.sort((x, y) => x.label.localeCompare(y.label));
        return out;
    }, [pages]);

    const go = (path) => dc.app.workspace.openLinkText(path, "", "tab");

    return <div className="home-areas">
        {areas.map(area => <div key={area.slug} className="home-area">
            <div className="home-area-head" onClick={() => setOpenArea(openArea === area.slug ? null : area.slug)}>
                <span>{openArea === area.slug ? "▾" : "▸"}</span>
                <span className="home-area-title">{titleOf(area.slug)}</span>
                <span className="td-count">{area.notes.length}</span>
            </div>
            {DESCRIPTIONS[area.slug] && <div className="home-area-desc">{DESCRIPTIONS[area.slug]}</div>}
            {openArea === area.slug && <div className="home-area-list">
                {area.index && <span className="home-link" onClick={() => go(area.index)}>Index</span>}
                {area.notes.map(n => <span key={n.path} className="home-link" onClick={() => go(n.path)}>{n.label}</span>)}
            </div>}
        </div>)}
    </div>;
}
```

---

## Protocols

```datacorejsx
// Notes under areas/protocols/. Fields read from their properties:
// protocol (active | paused | ended), started, ended, category.
const ROOT = "areas/protocols";
const pretty = (slug) => String(slug).split("-").map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(" ");
const toISO = (v) => v == null || v === "" ? null
    : v.toISODate ? v.toISODate()
    : String(v).slice(0, 10);
const todayISO = () => {
    const n = new Date();
    const p = (x) => String(x).padStart(2, "0");
    return `${n.getFullYear()}-${p(n.getMonth() + 1)}-${p(n.getDate())}`;
};
// Day 1 is the start date itself.
const dayOf = (started, today) => Math.round((Date.parse(today) - Date.parse(started)) / 86400000) + 1;

return function Protocols() {
    const pages = dc.useQuery(`@page and path("${ROOT}")`);
    const [tab, setTab] = dc.useState("current");
    const today = todayISO();

    const items = dc.useMemo(() => pages
        .filter(p => !String(p.$path).endsWith("_index.md"))
        .map(p => ({
            path: p.$path,
            name: pretty(String(p.$path).split("/").pop().replace(/\.md$/, "")),
            state: String(p.value("protocol") ?? "unset"),
            started: toISO(p.value("started")),
            ended: toISO(p.value("ended")),
            category: p.value("category") ? String(p.value("category")) : null,
        }))
        .sort((a, b) => a.state.localeCompare(b.state) || a.name.localeCompare(b.name)), [pages]);

    const current = items.filter(i => i.state !== "ended");
    const ended = items.filter(i => i.state === "ended");
    const list = tab === "current" ? current : ended;
    const go = (path) => dc.app.workspace.openLinkText(path, "", "tab");

    return <div className="td">
        <div className="td-tabs">
            {[["current", "Current", current.length], ["ended", "Ended", ended.length]].map(([id, label, count]) =>
                <span key={id} className={"td-tab" + (tab === id ? " is-active" : "")} onClick={() => setTab(id)}>
                    {label}<span className="td-count">{count}</span>
                </span>)}
        </div>
        <div>
            {list.map(item => <div key={item.path} className="td-row td-wrap">
                <span className={"td-dot" + (item.state === "active" ? " is-active" : item.state === "paused" ? " is-paused" : "")}></span>
                <span className="td-text home-link" onClick={() => go(item.path)}>{item.name}</span>
                {item.category && <span className="td-chip">{item.category}</span>}
                {item.state === "active" && item.started && item.started <= today &&
                    <span className="td-due" title={`Since ${item.started}`}>Day {dayOf(item.started, today)}</span>}
                {(item.started || item.ended) &&
                    <span className="td-due">{item.started ?? "?"} → {item.ended ?? "…"}</span>}
                <span className="td-chip">{item.state}</span>
            </div>)}
            {list.length === 0 && <div className="td-empty">Nothing here.</div>}
        </div>
    </div>;
}
```

