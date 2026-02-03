# Groundskeeper Implementation Plan

**Goal:** Build a client-side web app that views Lawnchair launcher backup files.

**Architecture:** Single HTML file with Alpine.js for reactivity, Tailwind for styling, JSZip for unpacking backups, sql.js for reading SQLite, and protobuf.js for parsing metadata. All processing client-side.

**Tech Stack:** Alpine.js, Tailwind CSS (CDN), JSZip, sql.js, protobuf.js

---

## Reference: Lawnchair Data Structures

### Backup Contents
- `info.pb` - BackupInfo protobuf (grid size, version, timestamps)
- `launcher.db` - SQLite with favorites table
- `screenshot.png` - Home screen preview
- `wallpaper.png` - Wallpaper (optional)
- `*.xml` - Preferences

### Favorites Table Schema
```sql
_id INTEGER PRIMARY KEY
title TEXT
intent TEXT              -- Package/activity info
container INTEGER        -- -100=desktop, -101=hotseat, or folder ID
screen INTEGER           -- Screen number (if container is desktop)
cellX INTEGER            -- X position in grid
cellY INTEGER            -- Y position in grid
spanX INTEGER            -- Width (for widgets)
spanY INTEGER            -- Height (for widgets)
itemType INTEGER         -- 0=app, 2=folder, 4=widget, 6=deep_shortcut
appWidgetProvider TEXT   -- Widget component name
icon BLOB                -- Custom icon bitmap
rank INTEGER             -- Position in folder/hotseat
```

### Item Types
- `0` = Application
- `2` = Folder
- `4` = Widget
- `6` = Deep Shortcut

### Container Types
- `-100` = Desktop (home screen)
- `-101` = Hotseat (dock)
- `> 0` = Folder ID (item is inside a folder)

---

## Task 1: Project Scaffold

**Files:**
- Create: `index.html`
- Create: `proto/lawnchair.proto`

**Step 1: Create the protobuf schema file**

Create `proto/lawnchair.proto`:
```protobuf
syntax = "proto3";

message Timestamp {
    int64 seconds = 1;
    int32 nanos = 2;
}

message GridState {
    string grid_size = 1;
    int32 hotseat_count = 2;
    int32 device_type = 3;
}

message BackupInfo {
    int32 lawnchair_version = 1;
    int32 backup_version = 2;
    Timestamp created_at = 3;
    int32 contents = 4;
    int32 preview_width = 5;
    int32 preview_height = 6;
    GridState grid_state = 7;
    bool preview_dark_text = 8;
}
```

**Step 2: Create base HTML with CDN dependencies**

Create `index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Groundskeeper - Lawnchair Backup Viewer</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.14.8/dist/cdn.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/sql.js@1.11.0/dist/sql-wasm.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/protobufjs@7.4.0/dist/protobuf.min.js"></script>
    <style>
        [x-cloak] { display: none !important; }
    </style>
</head>
<body class="bg-gray-100 min-h-screen">
    <div x-data="groundskeeper()" x-init="init()" class="min-h-screen">
        <!-- Loading state -->
        <div x-show="loading" x-cloak class="flex items-center justify-center h-screen">
            <div class="text-center">
                <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-green-600 mx-auto"></div>
                <p class="mt-4 text-gray-600">Loading...</p>
            </div>
        </div>

        <!-- Main app (placeholder) -->
        <div x-show="!loading" class="p-4">
            <h1 class="text-2xl font-bold text-gray-800">Groundskeeper</h1>
            <p class="text-gray-600">Lawnchair Backup Viewer</p>
            <p class="mt-4 text-sm text-gray-500">Libraries loaded successfully.</p>
        </div>
    </div>

    <script>
    function groundskeeper() {
        return {
            loading: true,
            sqlReady: false,

            async init() {
                // Initialize sql.js
                this.SQL = await initSqlJs({
                    locateFile: file => `https://cdn.jsdelivr.net/npm/sql.js@1.11.0/dist/${file}`
                });
                this.sqlReady = true;
                this.loading = false;
                console.log('Groundskeeper initialized');
            }
        }
    }
    </script>
</body>
</html>
```

**Step 3: Verify it loads**

Open `index.html` in a browser. Should see:
- Brief loading spinner
- "Groundskeeper" heading
- "Libraries loaded successfully" message
- No console errors

**Step 4: Commit**

```bash
git add index.html proto/lawnchair.proto
git commit -m "feat: project scaffold with CDN dependencies"
```

---

## Task 2: File Drop Zone

**Files:**
- Modify: `index.html`

**Step 1: Add drop zone UI**

Replace the main app div with:
```html
        <!-- Main app -->
        <div x-show="!loading" class="min-h-screen">
            <!-- Header -->
            <header class="bg-white shadow-sm">
                <div class="max-w-7xl mx-auto px-4 py-4 sm:px-6 lg:px-8">
                    <h1 class="text-2xl font-bold text-gray-800">Groundskeeper</h1>
                    <p class="text-sm text-gray-500">Lawnchair Backup Viewer</p>
                </div>
            </header>

            <!-- No file loaded - show drop zone -->
            <div x-show="!backup" class="max-w-2xl mx-auto mt-12 px-4">
                <div
                    @dragover.prevent="dragging = true"
                    @dragleave.prevent="dragging = false"
                    @drop.prevent="handleDrop($event)"
                    :class="dragging ? 'border-green-500 bg-green-50' : 'border-gray-300 bg-white'"
                    class="border-2 border-dashed rounded-lg p-12 text-center transition-colors"
                >
                    <svg class="mx-auto h-12 w-12 text-gray-400" stroke="currentColor" fill="none" viewBox="0 0 48 48">
                        <path d="M28 8H12a4 4 0 00-4 4v20m32-12v8m0 0v8a4 4 0 01-4 4H12a4 4 0 01-4-4v-4m32-4l-3.172-3.172a4 4 0 00-5.656 0L28 28M8 32l9.172-9.172a4 4 0 015.656 0L28 28m0 0l4 4m4-24h8m-4-4v8m-12 4h.02" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
                    </svg>
                    <p class="mt-4 text-lg text-gray-600">Drop your <code class="bg-gray-100 px-1 rounded">.lawnchairbackup</code> file here</p>
                    <p class="mt-2 text-sm text-gray-500">or</p>
                    <label class="mt-2 inline-block cursor-pointer">
                        <span class="px-4 py-2 bg-green-600 text-white rounded-lg hover:bg-green-700 transition-colors">
                            Browse files
                        </span>
                        <input type="file" class="hidden" accept=".lawnchairbackup,.zip" @change="handleFileSelect($event)">
                    </label>
                    <p class="mt-6 text-xs text-gray-400">Your backup stays on your device. Nothing is uploaded.</p>
                </div>
            </div>

            <!-- File loaded placeholder -->
            <div x-show="backup" class="max-w-7xl mx-auto mt-8 px-4">
                <p class="text-green-600">Backup loaded!</p>
            </div>
        </div>
```

**Step 2: Add state and handlers to Alpine component**

Update the `groundskeeper()` function:
```javascript
    function groundskeeper() {
        return {
            loading: true,
            sqlReady: false,
            dragging: false,
            backup: null,

            async init() {
                this.SQL = await initSqlJs({
                    locateFile: file => `https://cdn.jsdelivr.net/npm/sql.js@1.11.0/dist/${file}`
                });
                this.sqlReady = true;
                this.loading = false;
                console.log('Groundskeeper initialized');
            },

            handleDrop(event) {
                this.dragging = false;
                const file = event.dataTransfer.files[0];
                if (file) this.loadFile(file);
            },

            handleFileSelect(event) {
                const file = event.target.files[0];
                if (file) this.loadFile(file);
            },

            async loadFile(file) {
                console.log('Loading file:', file.name);
                this.loading = true;
                try {
                    // For now, just mark as loaded
                    this.backup = { name: file.name };
                } catch (error) {
                    console.error('Error loading backup:', error);
                    alert('Error loading backup: ' + error.message);
                } finally {
                    this.loading = false;
                }
            }
        }
    }
```

**Step 3: Test the drop zone**

- Open in browser
- Drag a file over - border should turn green
- Drop or select any file - should show "Backup loaded!"

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add file drop zone UI"
```

---

## Task 3: Parse Backup ZIP

**Files:**
- Modify: `index.html`

**Step 1: Implement ZIP parsing in loadFile**

Replace the `loadFile` method:
```javascript
            async loadFile(file) {
                console.log('Loading file:', file.name);
                this.loading = true;
                try {
                    const zip = await JSZip.loadAsync(file);

                    // List contents for debugging
                    const files = [];
                    zip.forEach((path, entry) => {
                        files.push({ path, dir: entry.dir, size: entry._data?.uncompressedSize || 0 });
                    });
                    console.log('Backup contents:', files);

                    // Extract what we need
                    const backup = {
                        name: file.name,
                        files: files
                    };

                    // Screenshot
                    const screenshotFile = zip.file('screenshot.png');
                    if (screenshotFile) {
                        const blob = await screenshotFile.async('blob');
                        backup.screenshot = URL.createObjectURL(blob);
                    }

                    // Wallpaper
                    const wallpaperFile = zip.file('wallpaper.png');
                    if (wallpaperFile) {
                        const blob = await wallpaperFile.async('blob');
                        backup.wallpaper = URL.createObjectURL(blob);
                    }

                    this.backup = backup;
                    console.log('Backup parsed:', backup);
                } catch (error) {
                    console.error('Error loading backup:', error);
                    alert('Error loading backup: ' + error.message);
                } finally {
                    this.loading = false;
                }
            }
```

**Step 2: Display extracted images**

Update the "File loaded" section:
```html
            <!-- File loaded -->
            <div x-show="backup" class="max-w-7xl mx-auto mt-8 px-4">
                <div class="bg-white rounded-lg shadow p-6">
                    <h2 class="text-lg font-semibold text-gray-800" x-text="backup?.name"></h2>

                    <div class="mt-4 grid grid-cols-1 md:grid-cols-2 gap-4">
                        <!-- Screenshot -->
                        <div x-show="backup?.screenshot">
                            <h3 class="text-sm font-medium text-gray-600 mb-2">Screenshot</h3>
                            <img :src="backup?.screenshot" class="rounded-lg border max-h-96 object-contain">
                        </div>

                        <!-- Wallpaper -->
                        <div x-show="backup?.wallpaper">
                            <h3 class="text-sm font-medium text-gray-600 mb-2">Wallpaper</h3>
                            <img :src="backup?.wallpaper" class="rounded-lg border max-h-96 object-contain">
                        </div>
                    </div>

                    <!-- Debug: file list -->
                    <details class="mt-4">
                        <summary class="text-sm text-gray-500 cursor-pointer">Backup contents</summary>
                        <pre class="mt-2 text-xs bg-gray-100 p-2 rounded overflow-auto" x-text="JSON.stringify(backup?.files, null, 2)"></pre>
                    </details>
                </div>
            </div>
```

**Step 3: Test with a real backup**

- Load a `.lawnchairbackup` file
- Should see screenshot and wallpaper images
- Should see file list in expandable details

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: parse backup ZIP and extract images"
```

---

## Task 4: Parse Protobuf Metadata

**Files:**
- Modify: `index.html`

**Step 1: Load and parse info.pb**

Add to the `loadFile` method after extracting images:
```javascript
                    // Parse info.pb
                    const infoFile = zip.file('info.pb');
                    if (infoFile) {
                        const infoData = await infoFile.async('arraybuffer');
                        backup.info = await this.parseBackupInfo(infoData);
                        console.log('BackupInfo:', backup.info);
                    }
```

**Step 2: Add parseBackupInfo method**

Add this method to the Alpine component:
```javascript
            async parseBackupInfo(data) {
                const root = await protobuf.load('proto/lawnchair.proto');
                const BackupInfo = root.lookupType('BackupInfo');
                const message = BackupInfo.decode(new Uint8Array(data));
                const info = BackupInfo.toObject(message, {
                    longs: Number,
                    defaults: true
                });

                // Parse grid size string (e.g., "5x7")
                if (info.gridState?.gridSize) {
                    const [cols, rows] = info.gridState.gridSize.split('x').map(Number);
                    info.gridState.cols = cols;
                    info.gridState.rows = rows;
                }

                // Convert timestamp
                if (info.createdAt?.seconds) {
                    info.createdAtDate = new Date(info.createdAt.seconds * 1000);
                }

                return info;
            },
```

**Step 3: Display backup info**

Add above the images grid:
```html
                    <!-- Backup Info -->
                    <div x-show="backup?.info" class="mt-4 grid grid-cols-2 sm:grid-cols-4 gap-4">
                        <div class="bg-gray-50 rounded p-3">
                            <p class="text-xs text-gray-500">Lawnchair Version</p>
                            <p class="font-medium" x-text="backup?.info?.lawnchairVersion"></p>
                        </div>
                        <div class="bg-gray-50 rounded p-3">
                            <p class="text-xs text-gray-500">Created</p>
                            <p class="font-medium" x-text="backup?.info?.createdAtDate?.toLocaleDateString()"></p>
                        </div>
                        <div class="bg-gray-50 rounded p-3">
                            <p class="text-xs text-gray-500">Grid Size</p>
                            <p class="font-medium" x-text="backup?.info?.gridState?.gridSize"></p>
                        </div>
                        <div class="bg-gray-50 rounded p-3">
                            <p class="text-xs text-gray-500">Dock Slots</p>
                            <p class="font-medium" x-text="backup?.info?.gridState?.hotseatCount"></p>
                        </div>
                    </div>
```

**Step 4: Test**

- Load a backup
- Should see version, date, grid size, dock slots
- Check console for parsed BackupInfo object

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: parse protobuf BackupInfo metadata"
```

---

## Task 5: Parse SQLite Database

**Files:**
- Modify: `index.html`

**Step 1: Load and query the database**

Add to `loadFile` after parsing info.pb:
```javascript
                    // Parse launcher.db
                    const dbFile = zip.file('launcher.db');
                    if (dbFile) {
                        const dbData = await dbFile.async('arraybuffer');
                        backup.layout = this.parseDatabase(dbData);
                        console.log('Layout:', backup.layout);
                    }
```

**Step 2: Add parseDatabase method**

```javascript
            parseDatabase(data) {
                const db = new this.SQL.Database(new Uint8Array(data));

                // Query favorites table
                const favoritesResult = db.exec(`
                    SELECT _id, title, intent, container, screen,
                           cellX, cellY, spanX, spanY, itemType,
                           appWidgetProvider, icon, rank
                    FROM favorites
                `);

                if (!favoritesResult.length) {
                    return { screens: [], dock: [], folders: {} };
                }

                const columns = favoritesResult[0].columns;
                const rows = favoritesResult[0].values;

                // Convert to objects
                const items = rows.map(row => {
                    const item = {};
                    columns.forEach((col, i) => item[col] = row[i]);

                    // Parse intent to get package name
                    if (item.intent) {
                        const match = item.intent.match(/component=([^;/]+)/);
                        if (match) item.packageName = match[1];
                    }

                    return item;
                });

                // Group by container
                const CONTAINER_DESKTOP = -100;
                const CONTAINER_HOTSEAT = -101;

                const screens = {};
                const dock = [];
                const folders = {};

                items.forEach(item => {
                    if (item.container === CONTAINER_DESKTOP) {
                        const screenNum = item.screen || 0;
                        if (!screens[screenNum]) screens[screenNum] = [];
                        screens[screenNum].push(item);
                    } else if (item.container === CONTAINER_HOTSEAT) {
                        dock.push(item);
                    } else if (item.container > 0) {
                        // Item is inside a folder
                        if (!folders[item.container]) folders[item.container] = [];
                        folders[item.container].push(item);
                    }
                });

                // Sort dock by rank/cellX
                dock.sort((a, b) => (a.rank || a.cellX) - (b.rank || b.cellX));

                // Attach folder contents to folder items
                items.filter(i => i.itemType === 2).forEach(folder => {
                    folder.contents = folders[folder._id] || [];
                    folder.contents.sort((a, b) => a.rank - b.rank);
                });

                db.close();

                return {
                    screens: Object.entries(screens)
                        .sort(([a], [b]) => Number(a) - Number(b))
                        .map(([num, items]) => ({ number: Number(num), items })),
                    dock,
                    allItems: items
                };
            },
```

**Step 3: Display basic layout stats**

Add after backup info grid:
```html
                    <!-- Layout Stats -->
                    <div x-show="backup?.layout" class="mt-4 grid grid-cols-2 sm:grid-cols-4 gap-4">
                        <div class="bg-blue-50 rounded p-3">
                            <p class="text-xs text-blue-600">Screens</p>
                            <p class="font-medium text-blue-800" x-text="backup?.layout?.screens?.length"></p>
                        </div>
                        <div class="bg-blue-50 rounded p-3">
                            <p class="text-xs text-blue-600">Dock Items</p>
                            <p class="font-medium text-blue-800" x-text="backup?.layout?.dock?.length"></p>
                        </div>
                        <div class="bg-blue-50 rounded p-3">
                            <p class="text-xs text-blue-600">Total Items</p>
                            <p class="font-medium text-blue-800" x-text="backup?.layout?.allItems?.length"></p>
                        </div>
                        <div class="bg-blue-50 rounded p-3">
                            <p class="text-xs text-blue-600">Folders</p>
                            <p class="font-medium text-blue-800" x-text="backup?.layout?.allItems?.filter(i => i.itemType === 2).length"></p>
                        </div>
                    </div>
```

**Step 4: Test**

- Load a backup
- Should see screen count, dock items, total items, folders
- Check console for full layout object

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: parse SQLite database and extract layout"
```

---

## Task 6: Sidebar Navigation (Desktop)

**Files:**
- Modify: `index.html`

**Step 1: Add currentView state**

Add to Alpine data:
```javascript
            currentView: 'info',  // 'info', 'screen-0', 'dock', 'wallpaper', 'settings'
```

**Step 2: Restructure layout with sidebar**

Replace the "File loaded" section with:
```html
            <!-- File loaded - Desktop layout -->
            <div x-show="backup" class="hidden sm:flex h-[calc(100vh-73px)]">
                <!-- Sidebar -->
                <aside class="w-56 bg-white border-r overflow-y-auto">
                    <nav class="p-4 space-y-1">
                        <button
                            @click="currentView = 'info'"
                            :class="currentView === 'info' ? 'bg-green-100 text-green-800' : 'text-gray-600 hover:bg-gray-100'"
                            class="w-full text-left px-3 py-2 rounded-lg text-sm font-medium transition-colors"
                        >
                            Backup Info
                        </button>

                        <div class="pt-2">
                            <p class="px-3 text-xs font-semibold text-gray-400 uppercase tracking-wider">Screens</p>
                            <template x-for="screen in backup?.layout?.screens" :key="screen.number">
                                <button
                                    @click="currentView = 'screen-' + screen.number"
                                    :class="currentView === 'screen-' + screen.number ? 'bg-green-100 text-green-800' : 'text-gray-600 hover:bg-gray-100'"
                                    class="w-full text-left px-3 py-2 rounded-lg text-sm transition-colors"
                                >
                                    <span x-text="'Screen ' + (screen.number + 1)"></span>
                                    <span class="text-xs text-gray-400 ml-1" x-text="'(' + screen.items.length + ')'"></span>
                                </button>
                            </template>
                        </div>

                        <button
                            @click="currentView = 'dock'"
                            :class="currentView === 'dock' ? 'bg-green-100 text-green-800' : 'text-gray-600 hover:bg-gray-100'"
                            class="w-full text-left px-3 py-2 rounded-lg text-sm transition-colors"
                        >
                            Dock
                            <span class="text-xs text-gray-400 ml-1" x-text="'(' + (backup?.layout?.dock?.length || 0) + ')'"></span>
                        </button>

                        <button
                            x-show="backup?.wallpaper"
                            @click="currentView = 'wallpaper'"
                            :class="currentView === 'wallpaper' ? 'bg-green-100 text-green-800' : 'text-gray-600 hover:bg-gray-100'"
                            class="w-full text-left px-3 py-2 rounded-lg text-sm transition-colors"
                        >
                            Wallpaper
                        </button>
                    </nav>
                </aside>

                <!-- Main content -->
                <main class="flex-1 overflow-y-auto p-6 bg-gray-50">
                    <!-- Info View -->
                    <div x-show="currentView === 'info'">
                        <h2 class="text-lg font-semibold text-gray-800 mb-4">Backup Information</h2>
                        <!-- Info content here -->
                        <div class="grid grid-cols-2 gap-4 max-w-md">
                            <div class="bg-white rounded-lg p-4 shadow-sm">
                                <p class="text-xs text-gray-500">Lawnchair Version</p>
                                <p class="font-medium" x-text="backup?.info?.lawnchairVersion"></p>
                            </div>
                            <div class="bg-white rounded-lg p-4 shadow-sm">
                                <p class="text-xs text-gray-500">Created</p>
                                <p class="font-medium" x-text="backup?.info?.createdAtDate?.toLocaleDateString()"></p>
                            </div>
                            <div class="bg-white rounded-lg p-4 shadow-sm">
                                <p class="text-xs text-gray-500">Grid Size</p>
                                <p class="font-medium" x-text="backup?.info?.gridState?.gridSize"></p>
                            </div>
                            <div class="bg-white rounded-lg p-4 shadow-sm">
                                <p class="text-xs text-gray-500">Dock Slots</p>
                                <p class="font-medium" x-text="backup?.info?.gridState?.hotseatCount"></p>
                            </div>
                        </div>

                        <!-- Screenshot preview -->
                        <div x-show="backup?.screenshot" class="mt-6">
                            <h3 class="text-sm font-medium text-gray-600 mb-2">Preview</h3>
                            <img :src="backup?.screenshot" class="rounded-lg border shadow-sm max-h-80 object-contain">
                        </div>
                    </div>

                    <!-- Screen View (placeholder) -->
                    <template x-for="screen in backup?.layout?.screens" :key="screen.number">
                        <div x-show="currentView === 'screen-' + screen.number">
                            <h2 class="text-lg font-semibold text-gray-800 mb-4" x-text="'Screen ' + (screen.number + 1)"></h2>
                            <p class="text-gray-500" x-text="screen.items.length + ' items'"></p>
                            <!-- Grid will go here -->
                        </div>
                    </template>

                    <!-- Dock View (placeholder) -->
                    <div x-show="currentView === 'dock'">
                        <h2 class="text-lg font-semibold text-gray-800 mb-4">Dock</h2>
                        <p class="text-gray-500" x-text="(backup?.layout?.dock?.length || 0) + ' items'"></p>
                    </div>

                    <!-- Wallpaper View -->
                    <div x-show="currentView === 'wallpaper'">
                        <h2 class="text-lg font-semibold text-gray-800 mb-4">Wallpaper</h2>
                        <img :src="backup?.wallpaper" class="rounded-lg shadow-lg max-w-full">
                    </div>
                </main>
            </div>

            <!-- Mobile layout (placeholder for now) -->
            <div x-show="backup" class="sm:hidden p-4">
                <p class="text-gray-500">Mobile view coming soon. Please use a larger screen.</p>
            </div>
```

**Step 3: Test**

- Load a backup
- Click sidebar items, views should switch
- Screens should be listed dynamically

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sidebar navigation for desktop"
```

---

## Task 7: Visual Grid Renderer

**Files:**
- Modify: `index.html`

**Step 1: Add getItemsForCell helper**

Add to Alpine component:
```javascript
            getItemAt(items, x, y) {
                return items.find(item =>
                    item.cellX === x &&
                    item.cellY === y &&
                    item.container === -100  // Desktop only
                );
            },

            getItemTypeLabel(itemType) {
                const types = { 0: 'App', 2: 'Folder', 4: 'Widget', 6: 'Shortcut' };
                return types[itemType] || 'Unknown';
            },

            getItemIcon(item) {
                // Return a CSS class for the icon placeholder
                if (item.itemType === 2) return '📁';
                if (item.itemType === 4) return '📦';
                return '📱';
            },
```

**Step 2: Create grid component**

Replace the screen view placeholder with:
```html
                    <!-- Screen View -->
                    <template x-for="screen in backup?.layout?.screens" :key="screen.number">
                        <div x-show="currentView === 'screen-' + screen.number">
                            <h2 class="text-lg font-semibold text-gray-800 mb-4" x-text="'Screen ' + (screen.number + 1)"></h2>

                            <!-- Grid -->
                            <div
                                class="inline-grid gap-1 bg-gray-200 p-2 rounded-lg"
                                :style="`grid-template-columns: repeat(${backup?.info?.gridState?.cols || 5}, minmax(0, 1fr))`"
                            >
                                <template x-for="row in (backup?.info?.gridState?.rows || 6)" :key="row">
                                    <template x-for="col in (backup?.info?.gridState?.cols || 5)" :key="col">
                                        <div
                                            class="w-16 h-16 bg-white rounded-lg flex items-center justify-center text-xs"
                                            :class="getItemAt(screen.items, col - 1, row - 1) ? 'ring-2 ring-green-400' : 'opacity-50'"
                                        >
                                            <template x-if="getItemAt(screen.items, col - 1, row - 1)">
                                                <div class="text-center p-1 overflow-hidden">
                                                    <span class="text-xl" x-text="getItemIcon(getItemAt(screen.items, col - 1, row - 1))"></span>
                                                    <p class="text-[10px] leading-tight truncate mt-1" x-text="getItemAt(screen.items, col - 1, row - 1)?.title || 'Untitled'"></p>
                                                </div>
                                            </template>
                                        </div>
                                    </template>
                                </template>
                            </div>

                            <!-- Items list -->
                            <div class="mt-6">
                                <h3 class="text-sm font-medium text-gray-600 mb-2">Items on this screen</h3>
                                <div class="bg-white rounded-lg shadow-sm divide-y">
                                    <template x-for="item in screen.items" :key="item._id">
                                        <div class="px-4 py-2 flex items-center gap-3">
                                            <span class="text-xl" x-text="getItemIcon(item)"></span>
                                            <div class="flex-1 min-w-0">
                                                <p class="font-medium truncate" x-text="item.title || 'Untitled'"></p>
                                                <p class="text-xs text-gray-500" x-text="item.packageName || getItemTypeLabel(item.itemType)"></p>
                                            </div>
                                            <span class="text-xs text-gray-400" x-text="`(${item.cellX}, ${item.cellY})`"></span>
                                        </div>
                                    </template>
                                </div>
                            </div>
                        </div>
                    </template>
```

**Step 3: Test**

- Load a backup
- Navigate to a screen
- Should see grid with items positioned
- Should see item list below

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add visual grid renderer for screens"
```

---

## Task 8: Widget Spanning

**Files:**
- Modify: `index.html`

**Step 1: Update grid to handle widget spans**

The current approach renders cell-by-cell which doesn't handle widgets that span multiple cells. We need a different approach - render items positioned absolutely within the grid.

Replace the grid in screen view with:
```html
                            <!-- Grid -->
                            <div
                                class="relative bg-gray-200 p-2 rounded-lg"
                                :style="`
                                    display: grid;
                                    grid-template-columns: repeat(${backup?.info?.gridState?.cols || 5}, 64px);
                                    grid-template-rows: repeat(${backup?.info?.gridState?.rows || 6}, 64px);
                                    gap: 4px;
                                `"
                            >
                                <!-- Empty cells for background -->
                                <template x-for="i in ((backup?.info?.gridState?.cols || 5) * (backup?.info?.gridState?.rows || 6))" :key="'cell-' + i">
                                    <div class="bg-white/50 rounded-lg"></div>
                                </template>

                                <!-- Items positioned on grid -->
                                <template x-for="item in screen.items" :key="item._id">
                                    <div
                                        class="absolute bg-white rounded-lg shadow-sm flex items-center justify-center p-1 ring-1 ring-gray-200 hover:ring-green-400 transition-shadow cursor-pointer"
                                        :style="`
                                            left: calc(8px + ${item.cellX} * 68px);
                                            top: calc(8px + ${item.cellY} * 68px);
                                            width: calc(${item.spanX || 1} * 64px + ${(item.spanX || 1) - 1} * 4px);
                                            height: calc(${item.spanY || 1} * 64px + ${(item.spanY || 1) - 1} * 4px);
                                        `"
                                        :title="item.title + (item.packageName ? ' (' + item.packageName + ')' : '')"
                                    >
                                        <div class="text-center overflow-hidden">
                                            <span class="text-xl" x-text="getItemIcon(item)"></span>
                                            <p
                                                class="text-[10px] leading-tight truncate mt-1"
                                                :class="(item.spanX || 1) > 1 ? 'max-w-[100px]' : 'max-w-[50px]'"
                                                x-text="item.title || (item.itemType === 4 ? 'Widget' : 'Untitled')"
                                            ></p>
                                        </div>
                                    </div>
                                </template>
                            </div>
```

**Step 2: Update getItemIcon for better widget display**

```javascript
            getItemIcon(item) {
                if (item.itemType === 2) return '📁';
                if (item.itemType === 4) {
                    // Try to identify widget type from provider
                    if (item.appWidgetProvider?.includes('weather')) return '🌤️';
                    if (item.appWidgetProvider?.includes('clock')) return '🕐';
                    if (item.appWidgetProvider?.includes('calendar')) return '📅';
                    if (item.appWidgetProvider?.includes('music')) return '🎵';
                    return '📦';
                }
                return '📱';
            },
```

**Step 3: Test**

- Load a backup with widgets
- Widgets should span multiple cells
- Hover should show item details

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: support widget spanning in grid"
```

---

## Task 9: Dock View

**Files:**
- Modify: `index.html`

**Step 1: Replace dock view placeholder**

```html
                    <!-- Dock View -->
                    <div x-show="currentView === 'dock'">
                        <h2 class="text-lg font-semibold text-gray-800 mb-4">Dock</h2>

                        <!-- Visual dock -->
                        <div class="inline-flex gap-2 bg-gray-200 p-3 rounded-2xl">
                            <template x-for="item in backup?.layout?.dock" :key="item._id">
                                <div
                                    class="w-14 h-14 bg-white rounded-xl flex items-center justify-center shadow-sm hover:shadow-md transition-shadow cursor-pointer"
                                    :title="item.title + (item.packageName ? ' (' + item.packageName + ')' : '')"
                                >
                                    <div class="text-center">
                                        <span class="text-2xl" x-text="getItemIcon(item)"></span>
                                    </div>
                                </div>
                            </template>
                        </div>

                        <!-- Dock items list -->
                        <div class="mt-6 max-w-md">
                            <h3 class="text-sm font-medium text-gray-600 mb-2">Dock Items</h3>
                            <div class="bg-white rounded-lg shadow-sm divide-y">
                                <template x-for="(item, index) in backup?.layout?.dock" :key="item._id">
                                    <div class="px-4 py-2 flex items-center gap-3">
                                        <span class="text-gray-400 text-sm w-4" x-text="index + 1"></span>
                                        <span class="text-xl" x-text="getItemIcon(item)"></span>
                                        <div class="flex-1 min-w-0">
                                            <p class="font-medium truncate" x-text="item.title || 'Untitled'"></p>
                                            <p class="text-xs text-gray-500 truncate" x-text="item.packageName || getItemTypeLabel(item.itemType)"></p>
                                        </div>
                                    </div>
                                </template>
                            </div>
                        </div>
                    </div>
```

**Step 2: Test**

- Load a backup
- Navigate to Dock view
- Should see visual dock and item list

**Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add dock view"
```

---

## Task 10: Folder Expansion

**Files:**
- Modify: `index.html`

**Step 1: Add folder modal state**

Add to Alpine data:
```javascript
            selectedFolder: null,
```

**Step 2: Add click handler to folder items**

Update the grid item div to handle folder clicks:
```html
                                        @click="item.itemType === 2 ? selectedFolder = item : null"
```

**Step 3: Add folder modal**

Add before closing `</div>` of the main app:
```html
        <!-- Folder Modal -->
        <div
            x-show="selectedFolder"
            x-cloak
            @click.self="selectedFolder = null"
            @keydown.escape.window="selectedFolder = null"
            class="fixed inset-0 bg-black/50 flex items-center justify-center z-50"
        >
            <div class="bg-white rounded-2xl shadow-xl max-w-sm w-full mx-4 overflow-hidden" @click.stop>
                <div class="p-4 border-b bg-gray-50">
                    <div class="flex items-center justify-between">
                        <h3 class="font-semibold text-gray-800" x-text="selectedFolder?.title || 'Folder'"></h3>
                        <button @click="selectedFolder = null" class="text-gray-400 hover:text-gray-600">
                            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
                            </svg>
                        </button>
                    </div>
                    <p class="text-xs text-gray-500 mt-1" x-text="(selectedFolder?.contents?.length || 0) + ' items'"></p>
                </div>
                <div class="p-4 max-h-80 overflow-y-auto">
                    <div class="grid grid-cols-4 gap-2">
                        <template x-for="item in selectedFolder?.contents" :key="item._id">
                            <div class="flex flex-col items-center p-2">
                                <span class="text-2xl" x-text="getItemIcon(item)"></span>
                                <p class="text-[10px] text-center mt-1 truncate w-full" x-text="item.title || 'Untitled'"></p>
                            </div>
                        </template>
                    </div>
                </div>
            </div>
        </div>
```

**Step 4: Test**

- Load a backup with folders
- Click a folder in the grid
- Modal should open showing folder contents
- Click outside or press Escape to close

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add folder expansion modal"
```

---

## Task 11: Mobile Layout

**Files:**
- Modify: `index.html`

**Step 1: Add mobile tab state**

Add to Alpine data:
```javascript
            mobileTab: 'info',  // 'info', 'screens', 'wallpaper'
            mobileScreenIndex: 0,
```

**Step 2: Replace mobile placeholder with full layout**

```html
            <!-- Mobile layout -->
            <div x-show="backup" class="sm:hidden flex flex-col h-[calc(100vh-73px)]">
                <!-- Tab bar -->
                <div class="bg-white border-b px-2 py-1 flex gap-1 overflow-x-auto">
                    <button
                        @click="mobileTab = 'info'"
                        :class="mobileTab === 'info' ? 'bg-green-100 text-green-800' : 'text-gray-600'"
                        class="px-3 py-1.5 rounded-full text-sm font-medium whitespace-nowrap"
                    >Info</button>
                    <button
                        @click="mobileTab = 'screens'"
                        :class="mobileTab === 'screens' ? 'bg-green-100 text-green-800' : 'text-gray-600'"
                        class="px-3 py-1.5 rounded-full text-sm font-medium whitespace-nowrap"
                    >Screens</button>
                    <button
                        @click="mobileTab = 'dock'"
                        :class="mobileTab === 'dock' ? 'bg-green-100 text-green-800' : 'text-gray-600'"
                        class="px-3 py-1.5 rounded-full text-sm font-medium whitespace-nowrap"
                    >Dock</button>
                    <button
                        x-show="backup?.wallpaper"
                        @click="mobileTab = 'wallpaper'"
                        :class="mobileTab === 'wallpaper' ? 'bg-green-100 text-green-800' : 'text-gray-600'"
                        class="px-3 py-1.5 rounded-full text-sm font-medium whitespace-nowrap"
                    >Wallpaper</button>
                </div>

                <!-- Content -->
                <div class="flex-1 overflow-y-auto p-4">
                    <!-- Info tab -->
                    <div x-show="mobileTab === 'info'">
                        <div class="grid grid-cols-2 gap-3">
                            <div class="bg-white rounded-lg p-3 shadow-sm">
                                <p class="text-xs text-gray-500">Version</p>
                                <p class="font-medium" x-text="backup?.info?.lawnchairVersion"></p>
                            </div>
                            <div class="bg-white rounded-lg p-3 shadow-sm">
                                <p class="text-xs text-gray-500">Created</p>
                                <p class="font-medium" x-text="backup?.info?.createdAtDate?.toLocaleDateString()"></p>
                            </div>
                            <div class="bg-white rounded-lg p-3 shadow-sm">
                                <p class="text-xs text-gray-500">Grid</p>
                                <p class="font-medium" x-text="backup?.info?.gridState?.gridSize"></p>
                            </div>
                            <div class="bg-white rounded-lg p-3 shadow-sm">
                                <p class="text-xs text-gray-500">Dock Slots</p>
                                <p class="font-medium" x-text="backup?.info?.gridState?.hotseatCount"></p>
                            </div>
                        </div>
                        <img x-show="backup?.screenshot" :src="backup?.screenshot" class="mt-4 rounded-lg shadow-sm w-full">
                    </div>

                    <!-- Screens tab -->
                    <div x-show="mobileTab === 'screens'">
                        <template x-if="backup?.layout?.screens?.length">
                            <div>
                                <!-- Screen selector -->
                                <div class="flex items-center justify-between mb-4">
                                    <button
                                        @click="mobileScreenIndex = Math.max(0, mobileScreenIndex - 1)"
                                        :disabled="mobileScreenIndex === 0"
                                        class="p-2 rounded-full hover:bg-gray-100 disabled:opacity-30"
                                    >
                                        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7"></path>
                                        </svg>
                                    </button>
                                    <span class="text-sm text-gray-600">
                                        Screen <span x-text="mobileScreenIndex + 1"></span> of <span x-text="backup?.layout?.screens?.length"></span>
                                    </span>
                                    <button
                                        @click="mobileScreenIndex = Math.min(backup?.layout?.screens?.length - 1, mobileScreenIndex + 1)"
                                        :disabled="mobileScreenIndex >= backup?.layout?.screens?.length - 1"
                                        class="p-2 rounded-full hover:bg-gray-100 disabled:opacity-30"
                                    >
                                        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"></path>
                                        </svg>
                                    </button>
                                </div>

                                <!-- Items list for current screen -->
                                <div class="bg-white rounded-lg shadow-sm divide-y">
                                    <template x-for="item in backup?.layout?.screens[mobileScreenIndex]?.items" :key="item._id">
                                        <div
                                            class="px-4 py-2 flex items-center gap-3"
                                            @click="item.itemType === 2 ? selectedFolder = item : null"
                                            :class="item.itemType === 2 ? 'cursor-pointer hover:bg-gray-50' : ''"
                                        >
                                            <span class="text-xl" x-text="getItemIcon(item)"></span>
                                            <div class="flex-1 min-w-0">
                                                <p class="font-medium truncate" x-text="item.title || 'Untitled'"></p>
                                                <p class="text-xs text-gray-500 truncate" x-text="item.packageName || getItemTypeLabel(item.itemType)"></p>
                                            </div>
                                            <template x-if="item.itemType === 2">
                                                <span class="text-xs text-gray-400" x-text="(item.contents?.length || 0) + ' items'"></span>
                                            </template>
                                        </div>
                                    </template>
                                </div>
                            </div>
                        </template>
                    </div>

                    <!-- Dock tab -->
                    <div x-show="mobileTab === 'dock'">
                        <div class="bg-white rounded-lg shadow-sm divide-y">
                            <template x-for="item in backup?.layout?.dock" :key="item._id">
                                <div class="px-4 py-2 flex items-center gap-3">
                                    <span class="text-xl" x-text="getItemIcon(item)"></span>
                                    <div class="flex-1 min-w-0">
                                        <p class="font-medium truncate" x-text="item.title || 'Untitled'"></p>
                                        <p class="text-xs text-gray-500 truncate" x-text="item.packageName || 'App'"></p>
                                    </div>
                                </div>
                            </template>
                        </div>
                    </div>

                    <!-- Wallpaper tab -->
                    <div x-show="mobileTab === 'wallpaper'">
                        <img :src="backup?.wallpaper" class="rounded-lg shadow-lg w-full">
                    </div>
                </div>
            </div>
```

**Step 3: Test on mobile viewport**

- Use browser dev tools to simulate mobile
- All tabs should work
- Screen navigation arrows should work
- Folder click should open modal

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add mobile layout with tabs"
```

---

## Task 12: Reset/Load New File

**Files:**
- Modify: `index.html`

**Step 1: Add reset button to header**

Update the header to show file name and reset button when a file is loaded:
```html
            <!-- Header -->
            <header class="bg-white shadow-sm">
                <div class="max-w-7xl mx-auto px-4 py-4 sm:px-6 lg:px-8 flex items-center justify-between">
                    <div>
                        <h1 class="text-2xl font-bold text-gray-800">Groundskeeper</h1>
                        <p class="text-sm text-gray-500" x-text="backup ? backup.name : 'Lawnchair Backup Viewer'"></p>
                    </div>
                    <button
                        x-show="backup"
                        @click="resetBackup()"
                        class="px-3 py-1.5 text-sm text-gray-600 hover:text-gray-800 hover:bg-gray-100 rounded-lg transition-colors"
                    >
                        Load different file
                    </button>
                </div>
            </header>
```

**Step 2: Add resetBackup method**

```javascript
            resetBackup() {
                // Revoke object URLs to prevent memory leaks
                if (this.backup?.screenshot) URL.revokeObjectURL(this.backup.screenshot);
                if (this.backup?.wallpaper) URL.revokeObjectURL(this.backup.wallpaper);

                this.backup = null;
                this.currentView = 'info';
                this.mobileTab = 'info';
                this.mobileScreenIndex = 0;
                this.selectedFolder = null;
            },
```

**Step 3: Test**

- Load a backup
- Click "Load different file"
- Should return to drop zone
- Load another backup, should work

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add reset/load new file button"
```

---

## Task 13: Error Handling

**Files:**
- Modify: `index.html`

**Step 1: Add error state**

Add to Alpine data:
```javascript
            error: null,
```

**Step 2: Update loadFile with better error handling**

```javascript
            async loadFile(file) {
                console.log('Loading file:', file.name);
                this.loading = true;
                this.error = null;

                try {
                    // Validate file extension
                    if (!file.name.endsWith('.lawnchairbackup') && !file.name.endsWith('.zip')) {
                        throw new Error('Please select a .lawnchairbackup file');
                    }

                    const zip = await JSZip.loadAsync(file);

                    // Validate it's a Lawnchair backup
                    if (!zip.file('launcher.db')) {
                        throw new Error('This doesn\'t appear to be a valid Lawnchair backup (missing launcher.db)');
                    }

                    // ... rest of parsing code ...
```

**Step 3: Add error display**

Add after the drop zone div:
```html
                <!-- Error display -->
                <div x-show="error" class="max-w-2xl mx-auto mt-4 px-4">
                    <div class="bg-red-50 border border-red-200 rounded-lg p-4 flex items-start gap-3">
                        <svg class="w-5 h-5 text-red-500 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4m0 4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path>
                        </svg>
                        <div>
                            <p class="font-medium text-red-800">Error loading backup</p>
                            <p class="text-sm text-red-600 mt-1" x-text="error"></p>
                        </div>
                        <button @click="error = null" class="ml-auto text-red-400 hover:text-red-600">
                            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
                            </svg>
                        </button>
                    </div>
                </div>
```

**Step 4: Update catch block**

```javascript
                } catch (error) {
                    console.error('Error loading backup:', error);
                    this.error = error.message;
                } finally {
                    this.loading = false;
                }
```

**Step 5: Test**

- Try loading a non-backup file
- Should show error message
- Dismiss button should work

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add error handling and validation"
```

---

## Task 14: README

**Files:**
- Create: `README.md`

**Step 1: Write README**

```markdown
# Groundskeeper

A client-side web app for viewing Lawnchair launcher backup files.

## Features

- 📱 View home screen layouts, dock, and folders
- 🖼️ Preview wallpaper and screenshots
- 📊 See backup metadata (version, grid size, creation date)
- 🔒 Privacy-first: everything runs in your browser, files never leave your device
- 📲 Mobile-friendly responsive design

## Usage

1. Visit [groundskeeper.example.com](https://groundskeeper.example.com) (or open `index.html` locally)
2. Drop your `.lawnchairbackup` file onto the page
3. Browse your backup contents

## Getting a Lawnchair Backup

In Lawnchair:
1. Open Lawnchair Settings
2. Go to Backup & Restore
3. Tap "Create Backup"
4. Share or save the `.lawnchairbackup` file

## Development

No build step required. Just open `index.html` in a browser.

### Dependencies (loaded via CDN)

- [Alpine.js](https://alpinejs.dev/) - Reactivity
- [Tailwind CSS](https://tailwindcss.com/) - Styling
- [JSZip](https://stuk.github.io/jszip/) - ZIP parsing
- [sql.js](https://sql.js.org/) - SQLite in browser
- [protobuf.js](https://protobufjs.github.io/protobuf.js/) - Protocol buffer parsing

## Privacy

All processing happens client-side in your browser. Your backup file is never uploaded anywhere. This is particularly important given the privacy concerns that drove many users from Nova Launcher to Lawnchair in the first place.

## License

MIT
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README"
```

---

## Task 15: Final Polish & Deploy Prep

**Files:**
- Modify: `index.html`

**Step 1: Add favicon (inline SVG data URL)**

Add to `<head>`:
```html
    <link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🌿</text></svg>">
```

**Step 2: Add meta tags for social sharing**

Add to `<head>`:
```html
    <meta name="description" content="View and explore Lawnchair launcher backup files in your browser. Privacy-first: files never leave your device.">
    <meta property="og:title" content="Groundskeeper - Lawnchair Backup Viewer">
    <meta property="og:description" content="View and explore Lawnchair launcher backup files in your browser.">
    <meta property="og:type" content="website">
```

**Step 3: Add footer**

Add before closing `</div>` of main app container:
```html
        <!-- Footer (shown when no backup loaded) -->
        <footer x-show="!backup" class="fixed bottom-0 inset-x-0 py-4 text-center text-xs text-gray-400">
            <a href="https://github.com/yourusername/groundskeeper" class="hover:text-gray-600">View on GitHub</a>
            <span class="mx-2">•</span>
            <span>Made for the Lawnchair community</span>
        </footer>
```

**Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add favicon, meta tags, and footer"
```

**Step 5: Create GitHub repo and push**

```bash
gh repo create groundskeeper --public --description "Lawnchair backup viewer - view your launcher backups in the browser" --source . --push
```

---

Plan complete and saved to `docs/plans/2026-02-03-groundskeeper-implementation.md`.

**Two execution options:**

1. **Subagent-Driven (this session)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

2. **Parallel Session (separate)** - Open new session in the project directory, batch execution with checkpoints

Which approach?