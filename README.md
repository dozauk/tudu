# tudu

A personal planning web app. Organise projects, tasks, milestones, and trip itineraries in one place.

## Running

Open `index.html` in any modern browser. No build step, no server, no dependencies.

Data is persisted in `localStorage` under the key `tudu_v1`. Use **Export** (sidebar) to save a backup JSON file and **Import** to restore it.

---

## Features

### Planners & Projects
- Multiple **planners** (e.g. Home, Work, Travel), each with an icon
- Two **project types**:
  - **Default** — deadline-based projects with milestones, tasks, and notes
  - **Trip** — departure/return dates, planning phase + live itinerary view

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
- **Timeline view** — vertical day columns (Outlook-style), full day visible without scrolling
- 8 event types: ✈️ Flight, 🏨 Hotel, 🚗 Car hire, 🚌 Transfer, 🎟️ Tour, 🍽️ Restaurant, 📅 Event, 📝 Note
- Events: title, start/end time, details, multiple links (labelled)
- **Live mode** — when a trip is in progress, itinerary tab opens by default with today's events surfaced

### Project Views
- **Grid view** — card per project with progress bar and next milestone
- **Full view** — wide tiles with inline task list, quick-add

### Export
- **JSON export/import** — full data backup, merge-by-ID on import
- **Markdown export** (per project) — printable project summary including milestones, tasks with sub-items, notes, and full itinerary

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
| `mockup.html` | Original static design mockup (reference only) |
| `TODO.md` | Roadmap and future feature backlog |
| `README.md` | This file |
| `ARCHITECTURE.md` | Technical internals for developers |
