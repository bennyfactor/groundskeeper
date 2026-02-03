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
