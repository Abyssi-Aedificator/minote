# minote

A minimal, offline-first note-taking app that runs entirely in your browser.

- **Single self-contained file** — everything in `index.html` (no build step, no dependencies)
- **Markdown editing** with live preview, slash commands, and a built-in renderer
- **Folders, tags, pinning, archive/trash** for organization
- **Resizable panel** — drag the divider between the note list and editor to resize; width persists across sessions
- **Command palette** — `Ctrl+K` to quickly open notes, folders, or toggle settings
- **Focus mode** — distraction-free typewriter view with paragraph highlighting
- **Plain-text mode** — toggle `txt` in the editor header to write raw markdown without rendering
- **Image paste** — paste images directly into notes (stored as data URLs in IndexedDB)
- **Dropbox sync** — optional sync through your own Dropbox App key (client-side OAuth2 PKCE, no backend)
- **PWA** — installable, works offline with a service worker
- **Dark/light theme** with 8 accent colours
- **Export/import** full backup as JSON

## Usage

Open `index.html` in a browser. Notes are persisted in IndexedDB and never leave your device unless you sync or export.

## Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl+K` / `⌘K` | Open command palette |
| `Escape` | Exit focus mode, close menus |
| `Enter` (title) | Jump to note body |
| `/` (body) | Open slash command menu |

## Tech

Vanilla JS, IndexedDB, CSS custom properties, service worker. Zero dependencies.
