# tudu — Future Roadmap

## Google Drive Integration

### Prerequisites
- App must be served over HTTP(S) — OAuth won't work from `file://`
  - Local: `npx serve .` or `python3 -m http.server 8080`
  - Deployed: GitHub Pages (free HTTPS, no backend needed)
- Google Cloud Console: create project, enable Drive API, create OAuth 2.0 Web Client ID,
  register authorised origin(s). Enter Client ID in tudu → Connect Google Drive.

### Drive folder structure
```
My Drive/
  tudu/
    index.json                  ← planner list + folder/file IDs (v2 format)
    {Planner Name} {icon}/      ← Drive folder — share this for planner-level access
      planner.json              ← full Planner object (projects, tasks, events…)
      {Project Name}/           ← Drive folder — share this for single-project access
        attachments/            ← Drive folder for uploaded files (Phase 3)
```

### Phase 1 — Auth + data sync  ✅ DONE
- OAuth via Google Identity Services (token flow, `drive.file` scope)
- Client ID entered once in settings modal, stored in `tudu_drive_config` localStorage key
- On sign-in: load from Drive (Drive is source of truth); fallback to localStorage offline
- On every `updateData()`: 800 ms debounced write to Drive
- Sync indicator in topbar; Google account shown in sidebar

### Phase 2a — Per-planner Drive files  ✅ DONE
- Data split from one `tudu-data.json` → `index.json` + per-planner `planner.json` files
- `index.json` maps planner IDs → folder/file IDs (avoids repeated Drive searches)
- Auto-migrates old v1 `tudu-data.json` on first sign-in; renames it as backup
- Each `driveSaveData()` writes all planner files + updated index

### Phase 2b — Folder management  ✅ DONE
- On planner create: `driveSaveData` already creates the folder automatically
- On planner rename: `driveRenamePlannerFolder()` — `PATCH /drive/v3/files/{folderId}` with `{ name: newName+icon }` — triggered in `savePlannerEdits()`
- On planner delete (owned): `drivePlannerFolderTrash()` — moves Drive folder to trash automatically
- On planner delete (shared): just removes from `d.planners` via `removeSharedPlanner()`
- Project subfolder + attachments deferred to Phase 3 (needed for file uploads)

### Phase 3 — File attachments
- "Attach file" button on task drawer + event modal
- Upload: HTML `<input type=file>` → Drive multipart upload to project `attachments/` folder
  → add `DriveAttachment { id, name, mimeType, webViewLink }` to task/event
- Google Picker (optional): `gapi.load('picker')` → select existing Drive files
  - Requires an API key (add to Drive settings modal alongside Client ID)
- Render attachments as chips with file icon + "Open in Drive ↗" link

### Phase 4 — Sharing
See "Sharing design" section below.

---

## Sharing Design

### Mechanism: Google Picker (stays within `drive.file` scope)
`drive.file` grants access to files "created or **opened via Picker** by the app".
Picker counts as "opening" — so shared folders become accessible without upgrading scope.

**User A (owner) shares:**
1. tudu creates `tudu/{Planner}/` folder
2. Share button → enter User B's email → `drive.permissions.create` on the planner folder
3. User B receives a standard Google Drive sharing notification

**User B (collaborator) joins:**
1. "Add shared planner" in sidebar → Google Picker opens (showing "Shared with me")
2. User B selects the shared planner folder
3. tudu reads `planner.json` from that folder → stores as `SharedPlannerRef` in their `index.json`
4. Planner appears in sidebar with 👥 badge
5. Reads/writes go directly to the shared file — no tudu server involved

### Data model additions
```
Root {
  planners: Planner[]              // owned planners (unchanged)
  sharedPlanners: SharedRef[]      // references to others' planners
}
SharedRef {
  folderId: string        // Drive folder ID (used to re-find planner.json)
  dataFileId: string      // planner.json file ID (cached after Picker)
  ownerEmail: string      // display only
  accessRole: 'reader' | 'writer'  // read from Drive permissions on load
  lastSynced: string      // ISO timestamp
  cachedPlanner: Planner  // local cache — refreshed on tab focus
}
```

### Sidebar sections
```
Planners              ← owned
  🏠 Home
  ✈️ Holiday 2026
──────────────────
Shared with me        ← SharedRef[] where cachedPlanner exists
  👥 Work Q2          (Alice — editor)
  👁 Budget Review    (Bob — viewer)
  + Add shared planner
```

### Viewer vs Editor
- On load: `drive.permissions.list(folderId)` → find own email → determine role
- Viewer: all inputs/buttons disabled; "read only" badge in planner header
- Editor: full interaction; writes go to shared `planner.json`

### Conflict resolution (v1: last-write-wins)
- Auto-refresh shared planners when tab regains focus (`visibilitychange` event)
- Manual ↺ refresh button on shared planner header
- "Last synced N min ago" timestamp shown
- If write returns 409: reload from Drive before retrying

### Build order for Phase 4
```
4a  Share button on planner header → invite by email (drive.permissions.create)  ✅ DONE
4b  Picker API + "Add shared planner" flow                                        ✅ DONE
4c  Viewer/editor enforcement + focus-refresh + last-synced display               ✅ DONE (partial — viewer writes silently fail, no input disabling yet)
4d  Collaborator list + revoke in share modal                                     ✅ DONE
```

### Scope note
`drive.file` handles everything above. Full `drive` scope (and Google security assessment)
would only be needed if users want to browse ALL their Drive files — not required here.

---

## Calendar Integration
- [ ] iCal export (.ics) as a first step
- [ ] Sync itinerary events with Google Calendar (via Google Calendar API)
- [ ] Two-way sync: changes in external calendar reflect back in tudu
- [ ] Per-event toggle: "add to calendar" / "remove from calendar"
- [ ] CalDAV support for generic calendar clients

## AI Assistant
- [ ] Persistent chat bar at the bottom of the app
- [ ] LLM with read access to all planner/project/task/itinerary data
- [ ] Propose edits (add tasks, update status, fill in event details, etc.)
- [ ] Human-in-the-loop approval: proposed changes shown as a diff before applying
- [ ] Context-aware suggestions (e.g. "you have 3 unbooked options for this task")

## Task Sub-items
- [ ] User-editable status labels (currently: Option / Maybe / Booked / No)
- [ ] Drag-and-drop reordering of sub-items

## Itinerary Improvements

### Multi-day hotel events
- [ ] Hotel event gets `checkInDate`, `checkOutDate` fields (replacing single-day `date`); check-in and check-out times stored separately (`checkInTime`, `checkOutTime`)
- [ ] Event modal: when type = `hotel`, show date range picker (check-in date / check-out date) and separate time fields for each; hide the normal single-day selector
- [ ] Data: hotel is stored once on the check-in day (or as a top-level span object) — on render it is **virtualised** into multiple itinerary appearances:
  - **Check-in day** → timed event block "🏨 Check in · [Hotel name]" at check-in time
  - **Intermediate days** → grey all-day banner "🏨 [Hotel name]" at the top of each day column (list view: before timed events; calendar view: full-width banner above the hour grid)
  - **Check-out day** → timed event block "🏨 Check out · [Hotel name]" at check-out time
- [ ] Editing: clicking any of the virtualised rows opens the same hotel edit modal
- [ ] Calendar view: all-day banners sit in a sticky header band above the time grid, similar to how Google Calendar shows multi-day events
- [ ] Export (Markdown + iCal): hotel appears as a single span entry with check-in/check-out dates

## General
- [ ] Responsive mobile UI
- [ ] Offline support / PWA
- [ ] Recurring tasks / events
- [ ] Notifications / reminders
