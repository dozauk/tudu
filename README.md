# tudu

A personal planning web app. Organise projects, tasks, milestones, and trip itineraries — including a live map view of your trip locations.

## Running

**Locally:** `python3 -m http.server 8080` then open `http://localhost:8080` (Google OAuth requires HTTP/HTTPS, not `file://`).

**Online:** deployed automatically to GitHub Pages on every push to `main`. The deployed git SHA is shown in the sidebar footer.

No build step, no server, no dependencies beyond a browser.

Data is persisted in `localStorage` under the key `tudu_v1`. Use **Export** (sidebar) to save a backup JSON file and **Import** to restore it.

---

## Features

### Planners & Projects
- Multiple **planners** (e.g. Home, Work, Travel), each with an emoji icon
- Two **project types**:
  - **Default** — deadline-based projects with milestones, tasks, and notes
  - **Trip** — departure/return dates, planning phase + live itinerary with map

### Task Management
- Tasks with checkbox completion, drag-to-reorder
- **Sub-items** — a list of options/considerations per task, each with a click-to-cycle status: Option → Maybe → Booked → No
- **Task notes** — free-text notes in an inline expandable drawer
- **Milestone linking** — tasks can be assigned to a milestone; view tasks grouped by milestone or as a flat list
- **Hide done** toggle to filter completed tasks
- **Itinerary linking** — a task can be linked to a placeholder itinerary event (two-way navigation)

### Milestones
- Named milestones with dates, displayed on a timeline
- Status indicators: upcoming, next, past

### Trip Itinerary
- Auto-generated day-by-day structure from departure → return
- **List view** — collapsible days with events
- **Timeline view** — vertical day columns with a sticky all-day strip above a scrollable time grid (default 7am–11pm, widens automatically if events fall outside those hours)
- **Map view** — all events with a location are geocoded and plotted on a Google Map; click a pin to see event details or jump to edit
- 8 event types: ✈️ Flight, 🏨 Hotel, 🚗 Car hire, 🚌 Transfer, 🎟️ Tour, 🍽️ Restaurant, 📅 Event, 📝 Note
- Events: title, start/end time, location, details, multiple labelled links
- **Multi-day span events** — hotel and car hire events accept a check-out/drop-off date; intermediate days show an all-day banner ("night 2 of 4"), the end day shows a timed checkout row
- **Live mode** — when a trip is in progress, itinerary tab opens by default with today's events surfaced

### File Attachments
- Attach files to tasks and events (requires Google Drive)
- Drag files onto the drop zone in the event modal, or click **+ Add file(s)** to browse
- Files upload to a project `attachments/` folder in Drive and can be linked to multiple tasks and events
- Linked files shown as chips; click to open in Drive

### Project Views
- **Grid view** — card per project with progress bar and next milestone
- **Full view** — wide tiles with inline task list, quick-add

### Export
- **JSON export/import** — full data backup, merge-by-ID on import
- **Markdown export** (per project) — printable project summary including milestones, tasks with sub-items, notes, and full itinerary

---

## Google Drive Sync

Connect Google Drive to sync data across devices, share planners, and attach files.

### Setup
1. Open **Settings** (sidebar footer) and enter your **OAuth 2.0 Client ID**
2. Optionally enter a **Google API Key** to enable the shared planner picker, map view, and geocoding
3. Click **Save** then **Sign in with Google**

See Settings → "How to get a Client ID" and "Required Google Cloud APIs" for step-by-step instructions.

### Sync behaviour
- **On sign-in / page load** — tudu checks the Drive modification time of each planner file. It only downloads planners that have been updated on Drive since your last sync — no unnecessary downloads.
- **On every edit** — changes are debounced and written to Drive 800 ms after the last keystroke.
- **On tab focus** — Drive modification times are rechecked. If another device has made changes they are pulled silently and the topbar shows "Updated from Drive".
- **Conflict** — if Drive is ahead *and* you have unsaved local changes, an amber banner appears: choose "Load from Drive" or "Keep my version".

### Sharing
- **Share a planner** — click the share icon on any planner in the sidebar, enter a collaborator's email, choose Editor or Viewer
- **Join a shared planner** — click "Add shared planner" in the sidebar, use the Google Picker to select the shared folder
- Editors can write; viewers see a read-only copy that auto-refreshes on tab focus

---

## Settings

Always accessible via the **⚙ Settings** button in the sidebar footer, or the gear icon in the Drive user row.

| Section | Contents |
|---|---|
| Google Drive | Live auth status, Client ID input, sign in/out |
| API Keys | Google API Key (Picker + Maps + Geocoding) — bring your own key |

---

## Keyboard & UX
- `Escape` closes any open modal
- Clicking outside a modal closes it **only if no data has been entered**
- Drag handle (⠿) on task rows to reorder

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire application — HTML, CSS, JS in one file |
| `privacy.html` | Privacy policy page |
| `img/tudu.svg` | App logo (SVG, used as favicon and sidebar icon) |
| `img/favicon.ico` | Favicon (ICO, multi-size) |
| `img/tudu-*.png` | Logo PNGs at 16, 32, 48, 64, 128, 256 px |
| `TODO.md` | Roadmap and future feature backlog |
| `README.md` | This file |
| `ARCHITECTURE.md` | Technical internals for developers |
| `.github/workflows/deploy.yml` | GitHub Actions — injects git SHA, deploys to Pages |
