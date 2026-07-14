# Project: USB Wiki App (index.html)

Single-file HTML/CSS/JS app for browsing and editing a Markdown-based wiki
stored in a folder on a USB drive (or any local folder). No build step —
everything lives in one `<script>` tag using vanilla JS. External libraries
loaded from CDN: `marked` (Markdown → HTML), `DOMPurify` (HTML sanitizing),
`localforage` (IndexedDB wrapper), `tinymce` (rich text editor for the
add/edit form).

## Core architecture — two data stores that must stay in sync

1. **On-disk files** (source of truth) — `.md` files inside
   `<selected-folder>/pages/<category>/`, accessed through the browser's
   File System Access API (`rootHandle`, `pagesDirHandle`). This is what
   persists across sessions and what a person can edit directly with a text
   editor from the USB drive.
2. **IndexedDB** (`db`, via `localforage`) — an in-browser cache of parsed
   page objects (`wikiPages` array), rebuilt from the files. This is what
   the UI actually reads from when rendering pages, navigation, and search.
   IndexedDB never contains anything the files don't also contain — it's a
   derived cache, not a second source of truth.

`syncFromFileSystem()` is the one function that reconciles the two: it reads
every `.md` file on disk, compares it against what's cached in IndexedDB,
and updates/creates/deletes IndexedDB records to match. Nothing should ever
write to IndexedDB without either (a) having just written the corresponding
file first, or (b) having just read the corresponding file in a sync pass.

## The Markdown ⇄ HTML pipeline (the "markup" layer)

The wiki content itself is always Markdown on disk and in IndexedDB — HTML
only exists transiently, for display or editing:

- **Down (Markdown → HTML, for display):** `renderMarkdown()` runs raw
  Markdown through `marked.parse()` then `DOMPurify.sanitize()` and injects
  the result into `#contentArea`. This is read-only, presentational HTML.
- **Down (Markdown → structured fields, for editing):** `parseMarkdown()`
  takes a raw `.md` string and splits it into `{ title, content, terms,
  related, changeDesc }` so the edit form can populate its title field, rich
  text editor, key-terms rows, and related-pages rows. `parseKeyTerms()` and
  `parseRelated()` are helpers it calls for the two sub-sections.
- **Up (structured fields → Markdown, for saving):** `buildMarkdown()` takes
  the form's current title/content/terms/related and re-serializes them
  into a single Markdown string in the exact format `parseMarkdown()`
  expects — this round-trip has to stay symmetric, or edits will silently
  lose data on the next edit pass.
- **Up (rich-text HTML → Markdown, inside the editor only):** TinyMCE edits
  content as HTML internally; `getEditorContent()` / `htmlToMarkdown()`
  convert that HTML back to Markdown before it's handed to
  `buildMarkdown()`. This is the one place actual HTML markup briefly exists
  outside of `renderMarkdown()`'s output, and a likely source of bugs if
  TinyMCE's HTML doesn't map cleanly back to Markdown.

IndexedDB and the `.md` files only ever store the Markdown string — never
the rendered HTML.

## Data model — a "page" object (as stored in IndexedDB)

```
{
  id: "slug",                  // = baseName, used as the IndexedDB key
  title: "Display Title",
  category: "radio-room",
  content: "full markdown of the latest version",
  version: 3,                  // number of the latest version
  filename: "slug-v3.md",      // actual filename of the latest version
  baseName: "slug",
  versions: [ { id, version, filename, content, changeDesc, timestamp }, ... ],
  lastModified: <timestamp>,
  contentHash: "<length>:<sha256 hex>"
}
```

On disk, this maps to: `slug.md` (always mirrors the latest version's
content — the "pointer" file) plus `slug-v2.md`, `slug-v3.md`, etc. for each
historical version.

## Function reference

### Setup / dependency check
| Function | Description |
|---|---|
| `checkDependencies()` | IIFE that runs immediately on page load; checks that `marked`, `DOMPurify`, `localforage`, `tinymce` all loaded from CDN, and shows a friendly error screen + throws if any are missing. |

### Markdown rendering
| Function | Description |
|---|---|
| `renderMarkdown(markdown)` | Converts a Markdown string to sanitized HTML for display in `#contentArea`. |
| `getLatestVersionContent(page)` | Returns the Markdown content of a page's most recent version (falls back to `page.content` if no version history). |

### IndexedDB
| Function | Description |
|---|---|
| `initDB()` | Creates the `localforage` instance (`USBWiki` DB, `pages` store) and does a test write/delete to confirm it works. |
| `resetDatabase()` | Confirms with the user, wipes every IndexedDB record, then calls `syncFromFileSystem(true)` to rebuild the cache from disk. Used to recover from a corrupted/desynced cache. |
| `loadPages()` | Reads every record out of IndexedDB into the in-memory `wikiPages` array, sorts it by category then title, re-renders the nav, and shows the current or first page. Called after almost every mutation. |

### File System Access (disk I/O)
| Function | Description |
|---|---|
| `selectFolder()` | Opens the native folder picker, finds or creates a `pages/` subfolder, **wipes IndexedDB**, discovers categories, ensures each category folder exists, then does a full `syncFromFileSystem(true)`. |
| `discoverCategories(dirHandle)` | Scans immediate subfolders of the pages directory; a subfolder counts as a category if it contains at least one `.md` file (falls back to listing all subfolders if none qualify). |
| `updateCategoryDropdown()` | Refreshes the `<select>` used in the add/edit page modal with the current `categories` list. |
| `getFileHandle(category, filename)` | Gets (creating if needed) a `FileSystemFileHandle` for a specific category/filename. |
| `readFile(category, filename)` | Reads a file's text content via `getFileHandle`. Returns `null` on any failure. |
| `writeFile(category, filename, content)` | Writes content to a file, with up to `CONFIG.maxRetries` retry attempts on failure. This is the only function that actually mutates files on disk. |
| `listFiles()` | Enumerates every `.md` file across all known categories, returning `{name, category, modTime, size}` for each. |

### Sync engine
| Function | Description |
|---|---|
| `syncFromFileSystem(force)` | The core reconciliation function — reads every file on disk (`listFiles` + `readFile`), diffs it against IndexedDB by content hash, and creates/updates/deletes IndexedDB records accordingly. New files become v1 pages; changed files add a new version entry. Files with no matching IndexedDB key that have disappeared get their record deleted. When `force` is true, also does an integrity pass ensuring `page.content` matches the latest version's content. Always ends by calling `loadPages()`. |
| `hashContent(content)` | Produces a `"<length>:<sha256-hex>"` fingerprint of a string (with a non-crypto fallback hash if `crypto.subtle` is unavailable), used by `syncFromFileSystem` to detect real content changes vs. no-op touches. |

### Page save / edit / version history
| Function | Description |
|---|---|
| `saveNewPage(title, category, content, changeDesc)` | Creates a brand-new page: writes `slug.md` to disk, builds the IndexedDB record with a single v1 version entry, saves it, and refreshes the view. If the slug already exists, writes it as the next version instead. |
| `updatePage(pageId, newContent, changeDesc)` | Saves an edit to an existing page: writes a new `slug-vN.md` file, rewrites `slug.md` as the updated "latest" pointer, appends a version entry to the IndexedDB record, and refreshes the view. |
| `rollbackVersion(pageId, versionNum)` | Takes an old version's content and re-saves it as a **new** version on top (doesn't delete history) — writes both the new versioned file and the pointer file, then updates IndexedDB. |
| `openHistory(pageId)` | Renders the version-history modal for a page, newest first, with a rollback button on every version except the current one. |

### Navigation / display
| Function | Description |
|---|---|
| `renderNav()` | Rebuilds the sidebar's page list, grouped by category, from `wikiPages`. |
| `showPage(page)` | Renders a specific page into `#contentArea` (via `renderMarkdown`), updates the breadcrumb, version badge, and history/edit button visibility. |
| `showWelcome()` | Renders the home/welcome screen with per-category page-count cards. |
| `getFolderIcon(category)` | Returns an emoji icon for known category names (`radio-room`, `front-counter`, `activations`, `general`), or a generic folder icon otherwise. |
| `updateFolderStatus(online)` | Updates the sidebar's connection indicator (dot color, folder name, enabled/disabled state of resync/reset buttons) based on whether a folder is connected. |
| `updateSyncStatus(status)` | Updates just the status dot/text to show `'syncing'` vs `'online'` state, without touching button enablement. |
| `filterNav(query)` | Client-side search: shows/hides sidebar nav items whose title matches the search box text. |

### Add/Edit form — Key Terms & Related Pages row builders
| Function | Description |
|---|---|
| `addTermRow(term, definition)` | Appends an editable Term/Definition row to the Key Terms section of the form. |
| `removeTermRow(btn)` | Removes a term row (refuses if it's the last remaining row). |
| `addRelatedRow(page)` | Appends an editable "related page title" row to the form. |
| `removeRelatedRow(btn)` | Removes a related-page row (refuses if it's the last remaining row). |

### Markdown build/parse (the up/down conversion layer — see pipeline section above)
| Function | Description |
|---|---|
| `buildMarkdown(title, content, terms, related, changeDesc)` | Serializes form data **up** into a single Markdown string: `# Title`, body content, optional `## Key Terms` table, and `## Related Pages` list (defaults to a Home link if no related pages given). |
| `parseMarkdown(content)` | Parses a raw `.md` string **down** into `{title, content, terms, related, changeDesc}` for populating the edit form. Also strips a leading `<!-- CHANGE: ... -->` comment if present. |
| `parseKeyTerms(lines)` | Helper: turns raw `| Term | Definition |` table lines into `{term, definition}` objects, skipping header/separator rows. |
| `parseRelated(lines)` | Helper: turns `- [Label](slug)` bullet lines into a plain array of labels, excluding any Home link. |

### Modals
| Function | Description |
|---|---|
| `openAddModal()` | Resets and opens the page modal in "create" mode (empty form, first category pre-selected). |
| `openEditModal(pageId)` | Opens the page modal in "edit" mode, pre-filled via `parseMarkdown(getLatestVersionContent(page))`. |
| `openModalA11y(id)` | Opens a modal and moves focus into it (accessibility: remembers what was focused before, focuses the first focusable element inside). |
| `closeModal(id)` | Closes a modal and restores focus to whatever was focused before it opened. |

### Rich text editor (TinyMCE)
| Function | Description |
|---|---|
| `initEditor(content)` | Initializes TinyMCE on `#pageContent` (or just re-sets its content if already initialized), converting the incoming Markdown to HTML for the editor to display. Handles pasted images by inlining them as base64 `<img>` tags. |
| `getEditorContent()` | Pulls the current HTML out of TinyMCE and converts it back to Markdown via `htmlToMarkdown()`. |
| `htmlToMarkdown(html)` | Converts TinyMCE's HTML output back into Markdown syntax (headings, bold/italic, lists, links, tables, etc.) so it can be handed to `buildMarkdown()`. |

### Save / notifications / validation
| Function | Description |
|---|---|
| `escapeHtml(text)` | Escapes `& < > " '` for safe interpolation into HTML strings built with template literals. |
| `handleSavePage()` | The Save button's click handler: reads all form fields, converts editor HTML to Markdown, builds the full Markdown via `buildMarkdown()`, then calls either `updatePage()` (editing) or `saveNewPage()` (creating). |
| `showNotification(msg, isError, type)` | Shows a temporary toast notification. |
| `runValidation()` | Builds a report modal listing every page's version count and whether it has Key Terms / Related Pages sections. |

## Flow 1 — Adding a new page (UI → disk → IndexedDB)

1. User clicks **+ Add Page** → `openAddModal()` resets the form and opens
   the modal.
2. User fills in title, category, body content (via TinyMCE), optional key
   terms and related pages, then clicks **Save**.
3. `handleSavePage()` fires:
   - Reads the title/category/change-description fields.
   - Calls `getEditorContent()` → `htmlToMarkdown()` to turn the rich-text
     body into Markdown.
   - Calls `buildMarkdown()` to assemble the full page Markdown string
     (title heading + body + Key Terms table + Related Pages list).
   - Since this is a new page (`isEditing` is false), calls
     `saveNewPage(title, category, fullContent, changeDesc)`.
4. `saveNewPage()`:
   - Derives the slug (`baseName`) from the title.
   - Calls `writeFile(category, "slug.md", fullContent)` — **this is the
     disk write**, via the File System Access API.
   - Builds the in-memory page object + a single v1 version entry.
   - Calls `db.setItem(baseName, page)` — **this is the IndexedDB write**.
   - Calls `loadPages()` to refresh `wikiPages`, re-render the nav, and
     display the new page.

Net effect: the file lands on disk first, then IndexedDB is updated to
match, then the UI re-renders from IndexedDB. No `syncFromFileSystem()` call
is needed for a page created this way, because the write and the cache
update happen in the same function.

## Flow 2 — Manually editing a `.md` file on the USB drive, then resyncing

1. User closes the browser (or just leaves it open) and edits a `.md` file
   directly on the USB drive with a text editor — e.g. changes the body
   text of `pages/radio-room/radio-check-procedure.md`.
2. Back in the app, user clicks **🔄 Resync** → the handler calls
   `syncFromFileSystem(true)`.
3. `syncFromFileSystem()`:
   - Re-runs `discoverCategories()` if categories aren't already known.
   - Calls `listFiles()` to enumerate every `.md` file on disk right now.
   - For each file: calls `readFile()`, strips any `<!-- CHANGE: -->`
     comment, and computes `hashContent()` of the cleaned body.
   - Looks up the matching IndexedDB record by `baseTitle` (the slug with
     any `-vN` suffix stripped).
     - **No existing record** → treated as a brand-new page: creates a
       fresh IndexedDB record with a single version entry.
     - **Existing record, hash differs from `page.contentHash`** (or the
       raw content differs) → content actually changed: bumps the version
       number, appends a new version entry, updates `page.content` and
       `page.contentHash`, and saves it back to IndexedDB.
     - **Existing record, hash matches** → no real change; at most updates
       `lastModified` metadata.
   - After processing every file, checks for **deletions**: any IndexedDB
     key that didn't show up in `existingFileKeys` this pass gets removed
     from IndexedDB.
   - If `force` is true, also does an integrity pass making sure
     `page.content` equals the latest version's content for every page.
   - Finally calls `loadPages()` to rebuild `wikiPages` from IndexedDB and
     re-render the UI.

Net effect: disk is the source of truth; a resync makes IndexedDB catch up
to whatever is currently on disk, using the content hash (not just
modification time) to decide whether something genuinely changed and
deserves a new version entry.

## Where to look for sync/desync bugs

Given the two flows above, likely trouble spots if IndexedDB and the USB
folder disagree:
- `hashContent()` hashing the *cleaned* content (after stripping the
  `<!-- CHANGE: -->` comment) while some other code path compares against
  *raw* content, or vice versa — a mismatch here causes false "changed"
  or false "unchanged" detections.
- The pointer-file pattern (`slug.md` always mirrors the latest version):
  any write path that updates `slug-vN.md` but fails to also update
  `slug.md` (see the `pointerSuccess` failure branches in `saveNewPage`,
  `updatePage`, `rollbackVersion`) will leave disk and the next sync
  briefly inconsistent — worth checking if the pointer write is ever
  skipped or silently fails.
- `selectFolder()` unconditionally **wipes all of IndexedDB** before
  syncing — if this is called again on an already-connected folder (e.g.
  browser permission re-prompt), it will look like everything reset.
- Version numbering in `syncFromFileSystem` derives the next version from
  `versions[versions.length - 1].version + 1`, while `saveNewPage` /
  `updatePage` compute it the same way but before the sync pass has run —
  if a manual disk edit and an in-app edit happen close together, these
  two version-numbering paths can race and produce duplicate version
  numbers or skipped ones.
