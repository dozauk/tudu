# tudu — Architecture

Single-file vanilla JS application. Everything lives in `index.html`: HTML structure, CSS custom properties, and JavaScript. No framework, no build tooling, no external JS dependencies (one Google Fonts import).

---

## File Layout (index.html)

```
<style>          CSS — custom properties, component styles (~700 lines)
<body>           Static HTML shells for sidebar, topbar, conflict banner, list view, detail view
<modals>         Modal markup for: new/edit planner, new/edit project, new/edit event,
                 import, share planner, settings
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
  ITINERARY      renderItinerary(), renderCalendarView(), renderMapView(), renderEventRow()
  SPAN EVENTS    getSpanEventsForDay(), renderSpanBanner(), renderSpanEndRow()
  EVENT CRUD     openAddEventModal(), saveEvent(), deleteCurrentEvent()
  PROJECT FILES  renderFilesSection(), handleFileUpload(), uploadFilesForEvent()
  MAP VIEW       geocodeAddress(), loadMapsApi(), renderMapView()
  PLANNER CRUD   createPlanner(), savePlannerEdits()
  PROJECT CRUD   saveProject(), deleteCurrentProject()
  EXPORT/IMPORT  exportProjectMarkdown(), exportData(), handleImportFile()
  MODAL HELPERS  openModal(), closeModal()
  GOOGLE DRIVE   driveLoadData(), driveSaveData(), refreshOwnedPlanners(), refreshSharedPlanners()
  SETTINGS       openSettings(), updateSettingsDriveStatus(), saveDriveClientId()
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
  _shared?: SharedMeta  // present only on shared (non-owned) planners
}

SharedMeta {
  folderId: string        // Drive folder ID
  dataFileId: string      // planner.json file ID
  accessRole: 'reader' | 'writer'
  lastSynced: string      // ISO timestamp — shown in sidebar
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
  days: Day[]             // sparse — only days with events are stored
  files: ProjectFile[]    // Drive-backed file attachments
  driveAttachFolderId?: string  // cached Drive folder ID for attachments/
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
  time: string | null     // 'HH:MM' — start time
  endTime: string | null  // 'HH:MM' — end time (same day, or checkout/drop-off time for span events)
  endDate: string | null  // ISO date — for multi-day events (hotel, car); stored on start day only
  location: string | null // free-text address/place name; geocoded for map view
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

ProjectFile {
  id: string
  name: string
  mimeType: string
  driveFileId: string     // Google Drive file ID
  webViewLink: string     // direct Drive URL
  size: number | null     // bytes
  uploadedAt: string      // ISO timestamp
  linkedTaskIds: string[] // task IDs this file is linked to
  linkedEventIds: { [eventId]: dateStr }  // event IDs this file is linked to
}
```

### Key invariants
- `days[]` is **sparse**: only days with at least one event are persisted. `getTripDays(p)` generates the full date range on read and merges saved data in.
- **Span events** (`endDate` set) are stored once on their start day. `getSpanEventsForDay(allDays, targetDate)` virtualises appearances on intermediate and end days at render time — no data duplication.
- `Task.linkedEventDate` is redundant with `Event` location but kept for O(1) lookup without scanning all days.
- Old events with `ev.link` (string) are handled by `getEventLinks(ev)` which normalises both formats.

---

## URL Navigation

Hash-based routing preserves position across refresh and produces shareable links.

```
#/p/{plannerId}                    list view for a specific planner
#/p/{plannerId}/proj/{projectId}   project detail (planning tab)
#/p/{plannerId}/proj/{projectId}/itinerary   project detail (itinerary tab)
```

Key functions:
```js
buildNavHash()    // derives hash string from current activePlannerId / activeProjectId / currentDetailTab
pushNavHash()     // history.pushState — used by showList() and openProject() (back button works)
replaceNavHash()  // history.replaceState — used by switchDetailTab() (tab flips don't pollute history)
applyNavHash()    // parses location.hash and calls showList() or openProject(); called from init()
                  // and on window 'popstate' event
```

`init()` calls `applyNavHash()` instead of `showList()` directly, so Drive data reloads (which re-call `init()`) also restore the last position.

---

## Google Drive

### File layout in Drive
```
My Drive/
  tudu/
    index.json           ← { version:2, planners:[{id, name, icon, folderId, dataFileId}] }
    {Planner Name} {🎒}/ ← Drive folder — share to grant planner-level access
      planner.json       ← full Planner object (all projects, tasks, events)
      attachments/       ← Drive folder for uploaded files (created on first upload)
```

### Drive state variables
| Variable | Purpose |
|---|---|
| `driveGisReady` / `driveGapiReady` | script load flags; `maybeDriveInit()` fires when both true |
| `driveTokenValid` | whether a live OAuth token is held |
| `driveLocallyDirty` | true while a local change is waiting for the 800 ms debounce to flush |
| `driveRootFolderId` | ID of the `tudu/` root folder |
| `driveIndexFileId` | ID of `index.json` |
| `drivePlannerMap` | `{ [plannerId]: { folderId, dataFileId } }` — avoids repeated Drive searches |
| `driveUserInfo` | `{ name, email, picture }` from Google userinfo endpoint |
| `driveTokenClient` | GIS `TokenClient` instance |
| `driveSyncTimer` | debounce handle for 800 ms write delay |

### `tudu_drive_config` localStorage key
Persists Drive configuration and sync state across page loads:
```js
{
  clientId: string,           // OAuth 2.0 Client ID
  apiKey: string,             // Google API Key (Picker, Maps, Geocoding)
  lastEmail: string,          // pre-fills account chooser on next sign-in
  rootFolderId: string,       // cached tudu/ folder ID
  indexFileId: string,        // cached index.json file ID
  plannerMap: { [id]: { folderId, dataFileId } },
  lastSuccessfulSync: string, // ISO timestamp of last successful driveSaveData()
}
```

### Load flow
```
sign-in → handleDriveTokenResponse
  → driveLoadData()
    → get/create tudu/ root folder
    → look for index.json
      ├── found (v2): driveReadAllPlanners()
      │     → read index.json to build drivePlannerMap
      │     → if no lastSuccessfulSync: pull ALL planner.json files (new device / first sign-in)
      │     → if lastSuccessfulSync exists: check modifiedTime per planner file (parallel)
      │           → pull only planners where modifiedTime > lastSuccessfulSync
      │           → skip download entirely if nothing is newer (localStorage is current)
      │     → merge pulled planners into localStorage → init()
      └── not found:
           ├── v1 tudu-data.json exists → set as localStorage, driveInitialPush(), rename v1 as backup
           └── first time → driveInitialPush() (push local data to Drive)
```

### Save flow
```
updateData(fn)
  → localStorage write
  → driveLocallyDirty = true
  → scheduleDriveSync()   (800 ms debounce)
    → driveSaveData()
      → for each owned planner: update planner.json (create folder+file if new planner)
      → update index.json
      → saveDriveConfig({ lastSuccessfulSync: now })
      → driveLocallyDirty = false
```

### Conflict resolution
On tab focus (`visibilitychange`) and after each sign-in, `refreshOwnedPlanners()` checks Drive `modifiedTime` per planner against `lastSuccessfulSync`:

```
modifiedTime > lastSuccessfulSync?
  NO  → local data is current; nothing to do
  YES → another device wrote since our last sync
          driveLocallyDirty?
            NO  → pull quietly, update localStorage directly (bypasses updateData to
                  avoid re-triggering sync), flash "Updated from Drive" in topbar
            YES → show amber conflict banner:
                  "Load from Drive" → clears pending timer, calls driveLoadData()
                  "Keep my version" → dismisses banner; pending sync fires and wins
```

The `modifiedTime` check is metadata-only (`files.get?fields=modifiedTime`) — no file content is downloaded unless a planner is actually newer.

### Sharing
**Owner shares:**
1. Share button on planner items in sidebar (when Drive signed in)
2. `openShareModal(plannerId)` → `drive.permissions.list` to load current collaborators
3. `shareWithUser()` → `drive.permissions.create` (type=user, role=writer|reader)
4. `revokeShare(folderId, permId)` → `drive.permissions.delete`

**Collaborator joins:**
1. "Add shared planner" button in sidebar
2. `openSharedPlannerPicker()` → Google Picker (requires API key)
3. User selects shared folder → `handlePickerResult()`
4. Reads `planner.json`, checks permissions for own role, stores as `_shared` planner

**`driveSaveData` routing:**
- Owned planners: written to their own files + listed in `index.json`
- Shared planners (writer): written to `pl._shared.dataFileId` directly; excluded from `index.json`
- Shared planners (reader): no write attempt

**Auto-refresh on tab focus:**  `refreshOwnedPlanners()` for owned, `refreshSharedPlanners()` for shared — both called from `visibilitychange`.

### File attachments
Files are uploaded to the project's `attachments/` Drive subfolder via resumable multipart upload (`driveUploadFileWithProgress`). Each uploaded file becomes a `ProjectFile` record in `project.files[]` and can be linked to multiple tasks and events. Linking is many-to-many: `file.linkedTaskIds[]` and `file.linkedEventIds{}`.

Uploading from the event modal (`uploadFilesForEvent`) writes directly to `localStorage` without going through `updateData` after the Drive write — this avoids triggering an immediate re-sync of data that was just received from Drive.

---

## Map View

The itinerary has three view modes: List, Timeline (calendar), and Map.

### Google Maps API loading
`loadMapsApi()` lazy-loads the Maps JS API script on first use, using a global `window._mapsReady` callback. Concurrent callers queue on the same promise. API key comes from `tudu_drive_config.apiKey`.

### Geocoding
`geocodeAddress(address)` calls the Google Geocoding REST API and caches results in `localStorage['tudu_geocode_cache']` (address → `{ lat, lng }`). Cached results persist across page loads to avoid redundant API calls.

### Map rendering
`renderMapView(p, days)` is async and self-contained:
1. Collects all events with a non-empty `location` field
2. Calls `loadMapsApi()` (no-op if already loaded)
3. Geocodes all locations in parallel (`Promise.all`)
4. Places `google.maps.Marker` pins using the project's colour
5. Fits bounds to all markers; single marker gets `zoom: 13`
6. Click on a marker opens an `InfoWindow` with event details and an Edit button

### Required Google Cloud APIs
All use the same API key stored in Settings:
- **Maps JavaScript API** — renders the map
- **Geocoding API** — converts `location` strings to coordinates
- **Picker API** — shared planner folder selection

---

## Settings

The Settings modal (`modal-settings`) is always accessible from the sidebar footer (⚙ Settings button) regardless of Drive auth state. When signed in, the Drive user row also shows a gear icon.

### Sections
- **Google Drive** — shows live connection status (signed in as / not signed in / not configured); Client ID input; How to get a Client ID guide
- **API Keys** — single Google API Key field used for Picker, Maps JS API, and Geocoding API; labelled "bring your own key (open source mode)"

`openSettings()` populates both inputs and calls `updateSettingsDriveStatus()` to render the current auth state inside the modal. `updateDriveUI()` also calls `updateSettingsDriveStatus()` whenever auth state changes, keeping the modal in sync if it is open.

---

## Store Abstraction

```js
function getData()        // returns parsed Root object from localStorage
function updateData(fn)   // getData → fn(data) → JSON.stringify → localStorage → scheduleDriveSync()
```

**These two functions are the only coupling to localStorage.** All Drive sync side-effects flow through `updateData`. Exception: Drive pull operations write to localStorage directly to avoid re-triggering an immediate write back to Drive.

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
| `itinView` | `'list'`\|`'calendar'`\|`'map'` | Itinerary view mode |
| `addEventDay` | string\|null | Day context for the event modal |
| `editingEventId` | string\|null | Event being edited |
| `pendingEventId` | string\|null | Pre-generated ID for a new event (used for file linking before save) |
| `addingFromTaskId` | string\|null | Task that triggered "Add to itinerary" |
| `selectedEvType` | string | Event type picker state |
| `modalLinks` | `Link[]` | Links being built in the event modal |
| `taskGroupMode` | `'all'`\|`'milestone'` | Task list grouping |
| `hideDoneTasks` | boolean | Filter completed tasks |
| `expandedTasks` | `Set<string>` | Task IDs with drawer open |
| `collapsedMsGroups` | `Set<string>` | Milestone group IDs collapsed in task view |
| `dragTaskId` | string\|null | Task being dragged |
| `openPotentialForms` | `Set<string>` | Task IDs with potential-slot date form open |
| `driveLocallyDirty` | boolean | True while a local change is waiting to sync to Drive |

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
        ├── [list]     → renderSpanBanner() + renderEventRow()  per day
        ├── [calendar] → renderCalendarView(p, days, ...)
        └── [map]      → renderMapView(p, days)   (async)
```

**Partial refresh pattern** — after mutating a single event:
```
refreshItinDay(dateStr)
  → calendar or map view, or day has span events, or today: renderItinerary(p)  (full re-render)
  → list view, non-span day: patch itin-events-${dateStr} innerHTML only
```

---

## Span Events (Multi-day)

Events with `endDate` set span multiple days. They are stored once on their start day. `getSpanEventsForDay(allDays, targetDate)` computes the role of the event on any given day:

| Role | Date | Rendered as |
|---|---|---|
| `start` | `day.date === targetDate && endDate > targetDate` | All-day banner in list view; all-day strip in calendar |
| `intermediate` | `day.date < targetDate && endDate > targetDate` | All-day banner ("night N of M") |
| `end` | `day.date < targetDate && endDate === targetDate` | Timed event row at `endTime` (check-out) |

In calendar view, span banners appear in the sticky all-day strip above the scrollable time grid.

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

```js
let CAL_START = 7;    // dynamic — recomputed each render from actual event times
let CAL_END   = 23;   // default 7am–11pm; widens if events fall outside
let CAL_HOURS = CAL_END - CAL_START;
const PX_PER_HR = 38; // chosen to fit full day on screen without vertical scroll
const COL_WIDTH = 168; // px per day column
```

`timeToY(timeStr)` converts `'HH:MM'` to a pixel offset from the top of the grid.

Event blocks are absolutely positioned within each day column. If `endTime` is set and `endDate` is not, block height = `timeToY(endTime) - timeToY(time)`; otherwise a 48 px minimum.

### Sticky all-day strip
The calendar wraps day headers and an all-day row in `.itin-cal-sticky-area` so both stick to the top as the time grid scrolls beneath them. Span event banners and untimed events appear in the all-day strip (one cell per day column), not in the scrollable grid.

### Scroll sync
Three separate scroll containers must stay in horizontal sync:
- `.itin-cal-hdr-scroll` — day name headers
- `.itin-cal-allday-scroll` — all-day event strip
- `.itin-cal-body-outer` — scrollable time grid (the scroll driver)

The body's `scroll` event manually sets `scrollLeft` on the other two.

---

## Deployment

Deployed to GitHub Pages via `.github/workflows/deploy.yml`. On every push to `main`:
1. `git rev-parse --short HEAD` produces a short SHA
2. `sed` replaces the `__VERSION__` placeholder in `index.html` with the SHA
3. The patched file is uploaded as the Pages artifact

The `__VERSION__` string appears in the sidebar footer and shows the deployed git SHA on the live site.

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
4. **New event field?** Add to modal HTML, `openAddEventModal` reset, `openEditEventModal` populate, `saveEvent` collect, `renderEventRow` display, `renderCalendarView` display, `exportProjectMarkdown`. If it should appear on the map, ensure it feeds into `geocodeAddress` or marker content.
5. **State that survives re-renders?** Use a module-level `Set` or variable (see `expandedTasks`, `collapsedMsGroups`). Re-populate from data on `openProject()`.

---

## Known Patterns / Gotchas

- **`updateData` always reads fresh** — never cache the result of `getData()` across an async boundary; always call inside `updateData(d => ...)`.
- **Drive pull bypasses `updateData`** — when pulling from Drive (on load or tab focus), write directly to `localStorage` to avoid re-triggering `scheduleDriveSync()`. Call `init()` or targeted render functions manually afterwards.
- **`getActiveProject()`** calls `getData()` on every invocation (no cache). Fine for localStorage, will need caching for a network backend.
- **Drag-and-drop** uses HTML5 native DnD API. `dragstart` fires on the handle element (not the row), so `event.stopPropagation()` is not needed.
- **Task drawer re-render** — `renderDetailTasks(p)` re-renders all tasks. `expandedTasks` Set preserves which drawers were open. Textarea `onblur` saves notes before re-render clears them.
- **Legacy `ev.link`** — old single-link events still in localStorage are transparently handled by `getEventLinks(ev)`. No migration script needed.
- **`CAL_START/END/HOURS` are `let`** — recomputed at the start of every `renderCalendarView` call from actual event times. Don't treat them as stable constants across render cycles.
- **`pendingEventId`** — pre-generated before the event modal opens so files can be linked to a new (unsaved) event. Cleared to `null` when editing an existing event, and replaced by `editingEventId` on save.
