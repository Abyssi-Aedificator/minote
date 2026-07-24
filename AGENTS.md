# minote

Single-file PWA note-taking app. All code lives in `index.html` (~3070 lines: CSS + HTML + JS in one file).

## Quick start

Open `index.html` in a browser. No build step, no dependencies, no server required.

## Syntax validation

No linter or test suite. The only automated check is JS syntax validation:

1. Extract `<script>` content via regex from `index.html`
2. Write to a temp file
3. Run `node --check <temp-file>`

PowerShell:
```powershell
$m = [regex]::Match((Get-Content 'index.html' -Raw), '(?s)<script>\s*(.*?)\s*</script>')
$m.Groups[1].Value | Out-File '$env:TEMP\check.js' -Encoding utf8
node --check '$env:TEMP\check.js'
```

Run this after every JS change.

## Files

- `index.html` — entire app (CSS in `<style>`, JS in `<script>`, HTML in `<body>`)
- `sw.js` — service worker; cache name `minote-v1`, update `CACHE` constant when assets change
- `manifest.json` — PWA manifest; references `icon.svg`
- `icon.svg` — app icon
- `README.md` — user-facing docs

## Versioning

- `APP_VERSION` constant (top of `<script>`) — current version string
- `DEV` constant — toggles dev banner; set to `false` before release
- `VERSIONS` array (JS) — changelog entries rendered by the changelog modal
- Commits: `Version X.Y.Z` for version bumps, descriptive messages for features/fixes
- The dev banner must never be mentioned in the changelog

## Data model

**IndexedDB** (`minote-db`, version 3) with two stores:
- `notes` (keyPath `id`): `{ id, title, body, tags, images, folderId, status, color, pinned, updated }`
- `folders` (keyPath `id`): `{ id, name }`

**Folder sentinel**: `activeFolderId === '__none__'` is a client-side filter for notes with `folderId === null`. Not stored in the note itself.

**Note status**: `'active'` (default), `'archived'`, `'trashed'`. Permanent delete only from trash view. "Clear all" moves active notes to trashed.

**Images**: Base64 data URLs stored in `n.images[]` as `{ id, dataUrl }`. Paste-only (no file picker). No size limits enforced.

## Architecture

**Pattern**: Single-writer — in-memory `notes`/`folders` arrays are the source of truth. IndexedDB is persistence. `renderIndex()` and `renderEditor()` re-render from memory.

**Editor**: `contentEditable` div. `n.body` (plain string) is the authoritative source. Edits intercepted via `beforeinput`, string updated, DOM re-rendered from string. Never trust the DOM for body content.

**Markdown rendering pipeline** (4 layers):
1. `mdEscape(s)` — HTML entity escaping (`&<>"`)
2. `renderBodyHTML(text)` — splits on `\n`, wraps lines in `<span class="lm-line" data-line="N">`, tracks fenced code state
3. `lmRenderLine(line, state)` — block-level: fenced code, headings, HR, blockquotes, task/unordered/ordered lists
4. `lmInline(s)` — inline: code, bold/italic/strikethrough, links. Uses null-byte sentinels to prevent nested regex issues

**Active line**: `.lm-mark` spans are `display:none` by default (hiding syntax). `.lm-line-active .lm-mark` shows syntax only on the cursor line.

**Slash commands**: Opened by `/` at line start or after whitespace. 14 items in `SLASH_ITEMS`. `$` in insert template marks cursor placement.

**Raw mode**: `rawMode = true` — body gets monospace font, content is `mdEscape(n.body)`, Enter inserts `\n`, paste is plain-text only, slash commands disabled.

**Boot sequence**: `openDB()` → load from IndexedDB → normalize notes → render UI → restore sync state → `dropboxHandleRedirect()` → `dropboxCheckForUpdates()` → `checkVersionUpdate()`.

## Dropbox sync

OAuth2 PKCE, no backend. Single file at `/minote-backup.json`.

**Upload**: Whole-state overwrite (`{ notes, folders }`). Deduped via `_lastUploadedState` string comparison.

**Download**: Non-merge = full replace (`clearAll()` + re-insert). Merge = id-based union, newer-wins conflict resolution.

**Auto-upload**: 10s debounce after edits. Auto-sync on launch if `dbx.autoSync` is true.

**Parser**: `dbxParseBackup()` handles both `{ data: { notes, folders } }` and legacy raw array format. `norm()` backfills missing fields (`body` from `text`, defaults for `title`, `tags`, etc.).

**Gotchas**:
- PKCE verifier in `sessionStorage` (`minote.dbx.verifier`) — lost if tab closes during redirect
- Requires HTTPS or localhost (`isSecureContext` check)
- `dismissedRemoteAt` prevents re-prompting for same remote timestamp
- Upload always overwrites (`mode: 'overwrite'`), not a diff

## Export/import

`doExport()` produces `minote-export.json`:
```json
{ "app": "minote", "version": 1, "exportedAt": "<ISO>", "data": { "notes": [...], "folders": [...] } }
```

`doImport()` uses the same `dbxParseBackup()` parser — accepts both envelope and legacy formats.

**Import is a full destructive replace** — clears all local data first. No merge option (merge is Dropbox-only).

## Settings / theme

**localStorage keys**:
| Key | Stores |
|---|---|
| `minote-theme` | `'dark'` / `'light'` / `'system'` |
| `minote-accent` | JSON `{main, dim}` |
| `minote-panel-w` | Left panel width (px) |
| `minote-version` | Last-seen `APP_VERSION` |
| `minote-uploaded-state` | Last-uploaded JSON snapshot |
| `minote.dropbox` | Full Dropbox state object |

**SessionStorage**: `minote.dbx.verifier` — PKCE code verifier (transient)

**CSS custom properties** (dark defaults):
| Property | Value | Purpose |
|---|---|---|
| `--bg` | `#15171c` | Page background |
| `--panel` | `#1b1e25` | Panels, topbar |
| `--panel-2` | `#20232b` | Hover states, secondary surfaces |
| `--line` | `#2c3038` | Borders |
| `--text` | `#e7e4da` | Primary text |
| `--text-dim` | `#8b8d92` | Secondary text |
| `--accent` | `#e8a23a` | Primary accent (set via JS) |
| `--accent-dim` | `#5c4a26` | Dim accent (set via JS) |
| `--danger` | `#c1502e` | Destructive actions |

Light theme (`:root.light`) overrides `--bg`, `--panel`, `--panel-2`, `--line`, `--text`, `--text-dim`. Does NOT override `--accent`/`--accent-dim`.

**`setTheme(mode)`**: Toggles `.light` class on `<html>`, updates `<meta name="theme-color">`, maps accent by name across palettes (e.g., "gold" dark → "gold" light).

**Accent palette**: 8 named colors (gold, green, blue, purple, red, teal, pink, orange), each with `main` and `dim` for dark and light. Applied via `applyAccent()` which sets inline style on `:root`.

**`prefers-color-scheme`**: A `<script>` in `<head>` runs before paint to prevent flash. A `change` listener re-calls `setTheme('system')` on OS theme change.

**Settings panel**: Built dynamically on each open via `innerHTML`. Three cards: appearance (theme + accent), backup (export/import/clear), Dropbox sync.

## Responsive layout

**Breakpoint**: `@media (max-width:760px)`

- Panels toggle full-screen via `.layout.show-editor` class (set by `updateMobileView()` as a side-effect of `renderEditor()`)
- `.panel-resize` (drag handle) is hidden
- Topbar wraps, search goes full-width on its own row
- Mobile back button (`<-`) becomes visible

## Note list

**`visibleNotes()` pipeline**: Filter by view → filter by folder → filter by search term (title, body, tags) → filter by imageFilter → sort (pinned first, then newest first).

**Search**: Case-insensitive `.includes()` on title, body, and tags. Cumulative with imageFilter.

**Swipe gestures**: Left = archive/restore, right = toggle pin. `swipeSuppress` flag (350ms) prevents stray click after swipe.

**View tabs**: `'active'`, `'archived'`, `'trashed'`. `setView()` resets `activeId` and re-renders.

## Service worker

Cache-first strategy. Cache name `minote-v1` (static). Cached assets: `.`, `index.html`, `manifest.json`, `icon.svg`, 3 font files.

**Gotcha**: Static cache name means changes to `index.html` require a hard refresh or SW update to appear. Registration silently swallows errors.

## Key gotchas

- **IME composition**: During `compositionstart`, body is NOT re-rendered (would break IME). Only re-render on `compositionend`.
- **`pendingBodyEdit` flag**: When `beforeinput` handles an edit, the subsequent `input` event is skipped to avoid double-processing.
- **Empty lines**: `setCaretOffset` injects zero-width space (`\u200B`) as caret anchor in empty inline elements.
- **`nodeToPlainText`**: Custom DOM→text extraction. Handles `<br>`, `<div>`, `<p>` as newlines. Strips zero-width chars.
- **Swipe gesture**: `swipeSuppress` flag (350ms) prevents stray click after swipe on mobile.
- **Cross-browser**: `-webkit-box-decoration-break: clone` for code blocks/quotes in WebKit/Blink.
- **Inline links**: Only `https?://` URLs are rendered as links. Non-http URLs are left as-is.
- **ID generation**: `Date.now().toString(36) + Math.random().toString(36).slice(2,7)` — collision risk under rapid creation but adequate for single-user.
- **Save debounce**: 400ms via `scheduleSave()`. Pending saves flushed on `beforeunload`/`visibilitychange`.
