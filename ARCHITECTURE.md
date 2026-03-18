# tudu — Architecture

Single-file vanilla JS application. Everything lives in `index.html`: HTML structure, CSS custom properties, and JavaScript. No framework, no build tooling, no external JS dependencies (one Google Fonts import).

---

## File Layout (index.html)

```
<style>          CSS — custom properties, component styles (~450 lines)
<body>           Static HTML shells for sidebar, topbar, list view, detail view
<modals>         Modal markup for: new/edit planner, new/edit project, new/edit event, import
<script>
  STORE          getData() / updateData()
  CONSTANTS      COLORS, PLANNER_ICONS, EVENT_TYPES
  STATE          Module-level let/const variables
  HELPERS        Date utils, trip helpers, formatting
  SIDEBAR        renderSidebar()
  LIST VIEW      showList(), renderGrid(), renderFullView()
  OPEN PROJECT   openProject(), switchDetailTab()
  DETAIL HEADER  renderDetailHeader()
  DETAIL TASKS   renderDetailTasks(), renderTaskWrap(), drawer functions
  MILESTONES     addMilestone(), deleteMilestone()
  ITINERARY      renderItinerary(), renderCalendarView(), renderEventRow()
  EVENT CRUD     openAddEventModal(), saveEvent(), deleteCurrentEvent()
  PLANNER CRUD   createPlanner(), savePlannerEdits()
  PROJECT CRUD   saveProject(), deleteCurrentProject()
  EXPORT/IMPORT  exportProjectMarkdown(), exportData(), handleImportFile()
  MODAL HELPERS  openModal(), closeModal()
  INIT           init()
```

---

## Data Model

Stored as a single JSON object in `localStorage['tudu_v1']`.

```
Root {
  planners: Planner[]
}

Planner {
  id: string          // uid()
  name: string
  icon: string        // emoji from PLANNER_ICONS
  projects: Project[]
}

Project {
  id: string
  type: 'default' | 'trip'
  color: string       // key into COLORS (indigo/violet/blue/teal/green/amber/orange/pink)
  name: string
  desc: string
  notes: string       // free-text project notes

  // Default projects
  deadline: string | null   // ISO date 'YYYY-MM-DD'

  // Trip projects
  departure: string | null  // ISO date
  returnDate: string | null // ISO date

  milestones: Milestone[]
  tasks: Task[]
  days: Day[]         // sparse — only days with events are stored
}

Milestone {
  id: string
  name: string
  date: string        // ISO date
}

Task {
  id: string
  text: string
  done: boolean
  notes: string               // free-text task notes
  milestoneId: string | null  // links to Milestone.id
  items: SubItem[]
  linkedEventId: string | null    // links to Event.id
  linkedEventDate: string | null  // ISO date of the day containing that event
  potentialSlot: { dateFrom: string, dateTo: string, note: string } | null
    // date range shown as dashed placeholder in itinerary; null once event is linked
}

SubItem {
  id: string
  text: string
  status: 'option' | 'considering' | 'booked' | 'rejected'
  link: string | null   // editable URL; shown as inline input in the sub-item row
}

Day {
  date: string        // ISO date — key for lookup
  events: Event[]
}

Event {
  id: string
  type: 'flight' | 'hotel' | 'car' | 'transfer' | 'tour' | 'restaurant' | 'event' | 'note'
  title: string
  time: string | null     // 'HH:MM'
  endTime: string | null  // 'HH:MM' — only if time is set
  details: string | null  // multi-line text
  links: Link[] | undefined
  linkedTaskId: string | null  // back-link to Task.id

  // Legacy (pre-multi-link) — handled transparently by getEventLinks()
  link: string | undefined
}

Link {
  id: string
  label: string       // e.g. 'Ticket', 'Booking ref'
  url: string
}
```

### Key invariants
- `days[]` is **sparse**: only days with at least one event are persisted. `getTripDays(p)` generates the full date range on read and merges saved data in.
- `Task.linkedEventDate` is redundant with `Event` location but kept for O(1) lookup without scanning all days.
- Old events with `ev.link` (string) are handled by `getEventLinks(ev)` which normalises both formats.

---

## Store Abstraction

```js
function getData()        // returns parsed Root object from localStorage
function updateData(fn)   // getData → fn(data) → JSON.stringify → localStorage
```

**These two functions are the only coupling to localStorage.** To migrate to a backend API, replace them with fetch calls returning/accepting the same Root shape. All other code is unaware of the storage layer.

---

## State Variables

| Variable | Type | Purpose |
|----------|------|---------|
| `activePlannerId` | string\|null | Currently selected planner |
| `activeProjectId` | string\|null | Currently open project |
| `viewMode` | `'grid'`\|`'full'` | List page view |
| `editingProjectId` | string\|null | Project being edited in modal |
| `editingPlannerId` | string\|null | Planner being edited in modal |
| `selectedColor` | string | Color picker state |
| `selectedIcon` | string | Icon picker state |
| `selectedProjType` | `'default'`\|`'trip'` | Type picker state |
| `currentDetailTab` | `'planning'`\|`'itinerary'` | Active tab in project detail |
| `itinView` | `'list'`\|`'calendar'` | Itinerary view mode |
| `addEventDay` | string\|null | Day context for the event modal |
| `editingEventId` | string\|null | Event being edited |
| `addingFromTaskId` | string\|null | Task that triggered "Add to itinerary" |
| `selectedEvType` | string | Event type picker state |
| `modalLinks` | `Link[]` | Links being built in the event modal |
| `taskGroupMode` | `'all'`\|`'milestone'` | Task list grouping |
| `hideDoneTasks` | boolean | Filter completed tasks |
| `expandedTasks` | `Set<string>` | Task IDs with drawer open |
| `collapsedMsGroups` | `Set<string>` | Milestone group IDs collapsed in task view |
| `dragTaskId` | string\|null | Task being dragged |
| `openPotentialForms` | `Set<string>` | Task IDs with potential-slot date form open |

---

## Rendering Architecture

The app re-renders by DOM manipulation, not a virtual DOM. Key pattern:

```
User action
  → updateData(fn)    // mutate localStorage
  → render*(p)        // re-render affected DOM region
```

**Render tree for a project detail:**
```
openProject(pid)
  ├── renderDetailHeader(p)      // header card, progress ring
  ├── renderDetailPlanning(p)
  │     ├── milestones timeline (inline HTML)
  │     ├── renderDetailTasks(p)
  │     │     └── renderTaskWrap(p, t, draggable)
  │     │           └── renderTaskDrawerContent(p, t)
  │     │                 └── renderSubItem(tid, item)
  │     └── notes textarea
  └── renderItinerary(p)
        ├── live banner
        ├── [list]     → renderEventRow(dayDate, ev)  ×N
        └── [calendar] → renderCalendarView(p, days, ...)
```

**Partial refresh pattern** — after mutating a single event:
```
refreshItinDay(dateStr)
  → if calendar view: renderItinerary(p)  // full re-render (cheap)
  → if list view: patch itin-events-${dateStr} innerHTML only
```

---

## Trip Logic

### Phase detection
```js
getTripPhase(p) → 'planning' | 'live' | 'done' | null
```
- `planning`: today < departure
- `live`: departure ≤ today ≤ returnDate
- `done`: today > returnDate

### Day generation
```js
getTripDays(p) → Day[]
```
Generates every date from departure→return, merging persisted `p.days[]` by date string. Days without events are ephemeral (not stored).

### Auto-navigation
When `openProject()` is called on a live trip, it defaults to the Itinerary tab.

---

## Calendar View

Constants (in `index.html`):
```js
CAL_START = 6      // 06:00
CAL_END   = 23     // 23:00
CAL_HOURS = 17
PX_PER_HR = 38     // chosen to fit full day on screen without vertical scroll
COL_WIDTH = 168    // px per day column
```

`timeToY(timeStr)` converts `'HH:MM'` to a pixel offset from the top of the grid.

Event blocks are absolutely positioned within each day column. If `endTime` is set, block height = `timeToY(endTime) - timeToY(time)`; otherwise defaults to 48px minimum.

**Header scroll sync** — the day header row uses `overflow:hidden` and its `scrollLeft` is manually synced from the body's `scroll` event, since two separate scroll containers can't natively stay in sync.

---

## Modal Close Guard

```js
document.querySelectorAll('.modal-bg').forEach(bg => {
  bg.addEventListener('click', e => {
    if (e.target !== bg) return;
    const dirty = [...bg.querySelectorAll('input, textarea')]
      .some(el => el.value.trim());
    if (dirty) return;        // don't close if user has entered data
    bg.classList.remove('open');
  });
});
```

---

## Export Formats

### JSON (`exportData`)
Full `Root` object as pretty-printed JSON. Import merges planners by ID (existing planners are not overwritten).

### Markdown (`exportProjectMarkdown`)
Per-project summary. Sections: metadata, milestones, tasks (optionally grouped by milestone, with sub-items and notes), project notes, itinerary (trips only). Helper `mdTask(t, lines)` renders a single task and its sub-items.

---

## CSS Architecture

All CSS uses custom properties defined on `:root`. Dark mode is handled entirely by `@media(prefers-color-scheme:dark)` overriding the same properties — no class toggling needed.

Key properties:
```css
--accent / --accent-h / --accent-lite / --accent-text   /* indigo brand */
--bg / --surface / --surface-2                           /* backgrounds */
--border / --border-2                                    /* borders */
--text / --text-2 / --text-3                             /* text hierarchy */
--green / --amber / --red  (+bg, +t variants)            /* status colours */
--sw: 248px   /* sidebar width */
--th: 56px    /* topbar height */
--r-sm/md/lg  /* border radii */
--sh-sm/md    /* box shadows */
```

Component CSS follows a flat BEM-like naming convention: `.itin-day-header`, `.task-expand-btn`, `.pft-tasks`, etc. No CSS modules or scoping.

---

## Adding a New Feature — Checklist

1. **Data change?** Update the model section above. Add field to `saveProject()` or `saveEvent()`. Check `exportProjectMarkdown()` and `exportData()`.
2. **New modal?** Add HTML modal shell, wire `openModal()`/`closeModal()`, handle dirty-close guard automatically.
3. **New project type?** Add to `type-toggle` HTML, `setProjectType()`, `getTripPhase()` equivalent, card renderer, detail renderer.
4. **New event field?** Add to modal HTML, `openAddEventModal` reset, `openEditEventModal` populate, `saveEvent` collect, `renderEventRow` display, `renderCalendarView` display, `exportProjectMarkdown`.
5. **State that survives re-renders?** Use a module-level `Set` or variable (see `expandedTasks`, `collapsedMsGroups`). Re-populate from data on `openProject()`.

---

## Known Patterns / Gotchas

- **`updateData` always reads fresh** — never cache the result of `getData()` across an async boundary; always call inside `updateData(d => ...)`.
- **`getActiveProject()`** calls `getData()` on every invocation (no cache). Fine for localStorage, will need caching for a network backend.
- **Drag-and-drop** uses HTML5 native DnD API. `dragstart` fires on the handle element (not the row), so `event.stopPropagation()` is not needed.
- **Task drawer re-render** — `renderDetailTasks(p)` re-renders all tasks. `expandedTasks` Set preserves which drawers were open. Textarea `onblur` saves notes before re-render clears them.
- **Legacy `ev.link`** — old single-link events still in localStorage are transparently handled by `getEventLinks(ev)`. No migration script needed.
