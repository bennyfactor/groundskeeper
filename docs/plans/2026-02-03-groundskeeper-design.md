# Groundskeeper - Lawnchair Backup Viewer

A client-side web app for viewing and exploring Lawnchair launcher backup files.

## Background

Nova Launcher's January 2026 acquisition by Instabridge led to aggressive ad tracking, driving users to Lawnchair. While Lawnchair has backup/restore functionality, there's no way to inspect backup contents without restoring them. Groundskeeper fills this gap.

## Goals

1. **Primary:** View Lawnchair backup files (`.lawnchairbackup`) in a browser
2. **Secondary:** Import Nova Launcher backups and convert to Lawnchair format

## Non-Goals

- Server-side processing (everything runs client-side for privacy)
- Full backup editing (view-only for v1, editing is future scope)
- Supporting other launchers beyond Nova migration

## Technical Approach

### Stack

No build step required. Single HTML file with CDN dependencies:

- **Alpine.js** - Reactivity and state management
- **Tailwind CSS** - Styling
- **JSZip** - Unpack backup files
- **sql.js** - Read SQLite databases in-browser
- **protobuf.js** - Parse Lawnchair's `info.pb` metadata

### Lawnchair Backup Format

`.lawnchairbackup` files are ZIP archives containing:

| File | Format | Contents |
|------|--------|----------|
| `info.pb` | Protocol Buffer | Backup metadata, grid state, version info |
| `launcher.db` | SQLite | Home screen layout, apps, folders, widgets |
| `*.xml` | XML | Shared preferences |
| `preferences.preferences_pb` | Protocol Buffer | Datastore preferences |
| `screenshot.png` | PNG | Home screen preview |
| `wallpaper.png` | PNG | Wallpaper (optional) |

### Nova Backup Format (for migration)

`.novabackup` files are ZIP archives containing:

| File | Format | Contents |
|------|--------|----------|
| `launcher.db` | SQLite | Apps, favorites, screens |
| `*_launcher_preferences.xml` | XML | Settings (grid size, dock, etc.) |

### Data Flow

```
User drops file
       ↓
  JSZip.loadAsync()
       ↓
  ┌────┴────┐
  ↓         ↓
info.pb   launcher.db
  ↓         ↓
protobuf  sql.js
  ↓         ↓
  └────┬────┘
       ↓
  Alpine.js state
       ↓
  Render UI
```

## User Interface

### Desktop/Tablet Layout (≥640px)

```
┌─────────────────────────────────────────────────────┐
│  Header: "Groundskeeper" + file picker              │
├─────────────────────────────────────────────────────┤
│  Sidebar          │  Main Content                   │
│  ─────────────    │  ───────────────                │
│  • Backup Info    │  Visual grid of selected screen │
│  • Screen 1       │                                 │
│  • Screen 2  ←    │    [app] [app] [folder]         │
│  • Screen 3       │    [widget 2x2]  [app]          │
│  • Dock           │    [app] [app] [app]            │
│  • Wallpaper      │                                 │
│  • Settings       │         ─── Dock ───            │
│                   │    [app] [app] [app] [app]      │
└─────────────────────────────────────────────────────┘
```

### Mobile Layout (<640px)

```
┌─────────────────────────┐
│     Groundskeeper       │
│    [Select file...]     │
├─────────────────────────┤
│ ┌─────┬───────┬───────┐ │
│ │Info │Screens│  ...  │ │  ← Tab bar
│ └─────┴───────┴───────┘ │
├─────────────────────────┤
│                         │
│  [app] [app] [folder]   │
│  [widget 2x2]   [app]   │
│  [app] [app]   [app]    │
│                         │
│      ─── Dock ───       │
│  [app] [app] [app]      │
│                         │
├─────────────────────────┤
│     ← Screen 1 of 3 →   │
└─────────────────────────┘
```

### Interactions

- Drag-and-drop file onto page to load
- Click sidebar/tab items to switch views
- Click folder to expand contents
- Click app to see details (package name, position)
- Swipe between screens on mobile

## Data Display

### Backup Info Panel

- Lawnchair version
- Backup creation date
- Grid dimensions

### Screen View

- Visual grid matching actual layout
- App icons with labels
- Folders (expandable)
- Widgets (rectangles showing dimensions and type)
- Dock pinned at bottom

### Wallpaper Panel

- Displays `wallpaper.png` if present
- "View full size" option

### Settings Panel

- Key preferences (icon size, dock enabled, search bar)
- Not exhaustive - just commonly-tweaked settings

## Nova Migration Feature

Separate section for importing Nova backups:

1. User uploads `.novabackup`
2. App parses and displays preview
3. Shows what transfers:
   - ✅ App positions, folder contents, dock layout
   - ⚠️ Widget positions (must re-add manually)
   - ❌ Gestures, Nova-specific settings
4. "Export as Lawnchair Backup" generates `.lawnchairbackup`

Includes caveat: "Widgets must be re-added manually. Review before restoring."

## File Structure

```
groundskeeper/
├── index.html          # Main app
├── app.js              # Core logic
├── proto/
│   └── lawnchair.proto # Protobuf schema
├── docs/
│   └── plans/
│       └── 2026-02-03-groundskeeper-design.md
└── README.md
```

## Privacy

All processing happens client-side. The backup file never leaves the user's browser. This is a key selling point given the privacy concerns driving Nova's exodus.

## Hosting

Static files deployable anywhere: GitHub Pages, Netlify, Vercel, or self-hosted.

## Future Scope (not v1)

- Edit backup contents and re-export
- Compare two backups
- Backup file validation/repair
- Support additional launchers
