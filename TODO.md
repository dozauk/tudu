# tudu — Future Roadmap

## Planned Features

### Google Drive Integration

#### Prerequisites
- App must be served over HTTP(S) — OAuth won't work from `file://`
  - Local: `npx serve .` or `python3 -m http.server 8080`
  - Deployed: GitHub Pages (free HTTPS, no backend needed)
- Google Cloud Console: create project, enable Drive API + Picker API, create OAuth 2.0 Web Client ID, register authorised origin(s)

#### Drive folder structure
```
My Drive/
  tudu/
    index.json                  ← planner list + folder IDs (written first on setup)
    {Planner Name} {icon}/      ← Drive folder — share this to give someone planner access
      {Project Name}/           ← Drive folder — share this for single-project access
        data.json               ← full Project object (tasks, milestones, events…)
        attachments/            ← Drive folder for uploaded files
```

#### Data model additions
```
Root       { driveRootFolderId? }
Planner    { driveFolderId? }
Project    { driveFolderId?, driveDataFileId?, driveAttachmentsFolderId? }
Task       { attachments?: DriveAttachment[] }
Event      { attachments?: DriveAttachment[] }
DriveAttachment { id, name, mimeType, webViewLink }
```

#### Phase 1 — Auth + data sync  ← START HERE
- Add `<script src="https://accounts.google.com/gsi/client">` + `https://apis.google.com/js/api.js`
- OAuth scope: `https://www.googleapis.com/auth/drive.file`
  (only sees files this app created; upgrade to `drive` scope later for full sharing)
- Settings modal (gear icon in sidebar): Google Client ID input, sign-in/out
- Sync indicator in topbar: ✓ Saved / ⟳ Saving… / ⚠ Offline / ● Not connected
- Load flow: signed in → `GET /files/{id}?alt=media` on index.json → load project data files → merge into Root; signed out → fall back to localStorage
- Save flow: wrap `updateData()` → `scheduleDriveSync()` with 500 ms debounce → write only changed project's `data.json`; if structure changed, rewrite `index.json`
- Re-finding root on new device: search `name = 'tudu' and mimeType = 'application/vnd.google-apps.folder' and 'root' in parents` then fall back to create

#### Phase 2 — Folder lifecycle
- On planner create/rename → create/rename Drive folder, store `driveFolderId`
- On project create → create project folder + `attachments/` subfolder, store both IDs
- On project/planner rename → `PATCH /files/{id}` to rename Drive folder
- On delete → offer to trash Drive folder (warn: Drive is now source of truth)

#### Phase 3 — File attachments
- "Attach file" button on task drawer + event modal
- Upload path: HTML `<input type=file>` → Drive multipart upload to project `attachments/` folder → add `DriveAttachment` to task/event
- Picker path (optional enhancement): `gapi.load('picker')` → select existing Drive files without uploading
- Render as chips: file icon + name + "Open ↗" link; download link for PDFs

#### Phase 4 — Sharing
- Scope upgrade: `drive.file` allows creating permissions on files the app created; reading files shared *by others* requires `drive` scope — prompt user to re-authorise when they try to open a shared planner
- Share button on planner (sidebar) + project detail header
  → modal: email input + role (Viewer / Editor) → `POST /files/{folderId}/permissions`
  → list current collaborators with revoke option
- Sharing model: share the project folder → other user gets `data.json` + attachments;
  share the planner folder → access to all projects within

#### Notes / gotchas
- `drive.file` scope means data is invisible on a new sign-in unless the root folder is re-discovered by name search — always persist `driveRootFolderId` in localStorage as a fallback hint
- Per-project `data.json` matches the existing `Project` object exactly — no format migration
- Drive write quota: ~10 req/s; 500 ms debounce keeps well within limits
- Offline: buffer pending writes in a `pendingSync` queue; flush on reconnect (listen to `navigator.onLine`)

### Calendar Integration
- [ ] Sync itinerary events with Google Calendar (via Google Calendar API)
- [ ] CalDAV support for generic calendar clients (Apple Calendar, Outlook, etc.)
- [ ] Two-way sync: changes in external calendar reflect back in tudu
- [ ] Per-event toggle: "add to calendar" / "remove from calendar"
- [ ] iCal export (.ics) as a simpler first step

### AI Assistant
- [ ] Persistent chat bar at the bottom of the app
- [ ] LLM with read access to all planner/project/task/itinerary data
- [ ] Propose edits (add tasks, update status, fill in event details, etc.)
- [ ] Human-in-the-loop approval: proposed changes shown as a diff before applying
- [ ] Context-aware suggestions (e.g. "you have 3 unbooked options for this task")

### Backend & Multi-user
- [ ] REST/GraphQL API backend (swappable with current localStorage layer)
- [ ] User accounts and authentication
- [ ] Shared planners with collaborator access
- [ ] Real-time sync across devices/clients

### File Storage & Attachments
- [ ] Files linked to specific events or tasks with a label
- [ ] Google Drive integration: auto-create a folder per project, upload files (PDFs, tickets, etc.) through the tudu interface
- [ ] Preview PDFs inline (using browser PDF viewer or Google Drive embed)
- [ ] Support other providers: Dropbox, OneDrive

### Task Sub-items
- [ ] User-editable status labels (currently: Option / Maybe / Booked / No)
- [ ] Drag-and-drop reordering of sub-items

### General
- [ ] Responsive mobile UI
- [ ] Offline support / PWA
- [ ] Recurring tasks / events
- [ ] Notifications / reminders
