# The Dashboard UI

The dashboard is a three-panel browser interface built entirely in vanilla JavaScript -- no framework, no build step, no bundler. The frontend consists of `src/ui/index.html` (layout and modals), `src/ui/app.js` (~2,245 lines of application logic), and `src/ui/style.css` (theming with oklch colors). It communicates with the Node.js backend via a REST API defined in `src/server.mjs`, and uses SortableJS loaded from CDN for drag-and-drop.

## Three-Panel Layout

The layout divides the viewport into three resizable panels separated by draggable dividers:

1. **Left Sidebar** (`#sidebar`): A scope tree with expandable nodes showing category counts per scope. Contains the search input, collapse-all button, theme toggle, export button, and footer links. During drag operations, the sidebar collapses to a compact scope-tree-only view to serve as a drop target.

2. **Main Content** (`#mainContent`): Displays the selected scope's items. The content header shows the scope name, inheritance chain, and a context budget button. A filter bar with category pills lets users toggle visibility by category. Items are grouped by category with sort controls (name, size, date).

3. **Right Detail Panel** (`#detailPanel`): Shows metadata for the selected item -- scope, type, description, size, dates, and file path. Includes a file preview, "Claude Code Prompt" action buttons, and move/delete/open controls. This panel alternates with the **Context Budget Panel** (`#ctxBudgetPanel`), which displays token usage as a visual bar chart.

The panels are separated by draggable resizers (`#resizerLeft`, `#resizerRight`) implemented in `setupResizer()` (app.js lines 397-438), allowing users to adjust panel widths.

## REST API Routes

The backend in `src/server.mjs` (lines 102-646) exposes these endpoints:

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/scan` | Full scan of all scopes and items |
| GET | `/api/context-budget?scope=&limit=` | Token budget breakdown |
| GET | `/api/destinations?path=&category=&name=` | Valid move destinations for an item |
| GET | `/api/file-content?path=` | File content for detail panel preview |
| GET | `/api/session-preview?path=` | Parse JSONL session into readable conversation |
| GET | `/api/browse-dirs?path=` | List subdirectories for folder picker |
| GET | `/api/version` | Check for npm updates |
| POST | `/api/move` | Move item between scopes |
| POST | `/api/delete` | Delete an item |
| POST | `/api/restore` | Restore a deleted file (for undo) |
| POST | `/api/restore-mcp` | Restore a deleted MCP server entry (for undo) |
| POST | `/api/export` | Export all items to `~/.claude/exports/cco-backup-<timestamp>/` |

All file operations are guarded by `isPathAllowed()` (lines 45-52), which validates that paths stay under `~/.claude/` or the user's HOME directory, preventing path traversal attacks.

## SortableJS Drag-and-Drop

**Initialization:** SortableJS v1.15.6 is loaded from `cdn.jsdelivr.net`. `initSortable()` (app.js lines 1310-1378) creates Sortable instances on all `.sortable-zone` elements. Each zone has a `data-group` attribute matching a category group (memory, skill, mcp, etc.), and items with the same group can be dragged between zones.

**Drag flow:**
1. `onStart`: stores the dragged item, adds a `drag-active` class to the sidebar, and collapses it to scope-tree-only view.
2. The user drags the item to a different scope's sortable zone or drops it on a sidebar scope block.
3. `onEnd`: if the source and destination zones differ, shows the drag confirm modal via `showDragConfirm()`.
4. On confirmation, `doMove()` calls `POST /api/move`.
5. After the move completes, `refreshUI()` re-scans and re-renders everything.

**Sidebar drop zones** (lines 1380-1444): separate `dragover`/`drop` handlers on the document allow dropping items directly onto sidebar scope blocks, not just sortable zones. A `.drop-target` CSS class provides visual highlighting during hover.

**Locked item behavior**: when a locked item (config, hook, plugin) is dragged to a scope block, instead of calling the move API, the UI generates a Claude Code prompt and copies it to the clipboard (lines 1413-1421). This lets users paste the prompt into Claude Code to perform the operation manually.

## Search and Filter

**Search** (app.js lines 145-151): filters items by matching the query against a concatenation of `[name, description, category, subType, path].join(" ")` (lines 2143-2153). The search also filters sidebar scope visibility, hiding scopes with no matching items.

**Category filters** (app.js lines 196-221): toggle pills in the filter bar. An `activeFilters` Set tracks which categories are visible. Items must match at least one active filter; when no filters are active, all items are shown.

## Undo System

The dashboard supports undoing both moves and deletes:

**Move undo** (app.js lines 1835-1858): after a successful move, the UI creates an `undoFn` that calls `POST /api/move` in reverse, moving the item back to its original scope. A toast notification with an "Undo" button appears for 8 seconds.

**Delete undo** (app.js lines 1869-1952): before deleting, the UI reads the file content as a `backupContent` string. For MCP items, it stores `{ name, config, mcpJsonPath }` as `mcpBackup`. After deletion, the `undoFn` calls either `POST /api/restore` (for files) or `POST /api/restore-mcp` (for MCP entries). For skills (directories), restoration creates the directory and writes `SKILL.md` from the backup.

**Toast notifications** (app.js lines 1961-1983): appear at the bottom of the viewport. Support error styling, undo buttons, persistent close buttons, and auto-hide (4 seconds default, 8 seconds when an undo action is available).

## Additional Features

**Bulk operations** (lines 311-357): a checkbox select mode enables multi-select. Bulk move (restricted to items of the same category) and bulk delete operate on the selection.

**Claude Code Prompt buttons** (`renderCcActions()`, lines 811-896): category-specific prompts that users copy and paste into Claude Code. Actions include "Explain This", "Edit", "Resume Session", "Modify", and "Remove". Each prompt is multi-step, asking Claude to read the item, explain it, and confirm before acting.

**Theme toggle**: switches a `.dark` class on the document body (line 364). CSS uses the `oklch` color space with separate light and dark variable sets for consistent color rendering.

**Export** (lines 913-939): calls `POST /api/export`, which copies all scanned items to `~/.claude/exports/cco-backup-<timestamp>/` organized by `scopeId/category/`.

## Why This Design Matters

The vanilla JS architecture with zero build tooling means the dashboard loads instantly from the Node server with no compilation step -- the same files developers can read in their editor are exactly what runs in the browser. SortableJS provides production-grade drag-and-drop without framework coupling. The undo system, which backs up content before destructive operations, gives users confidence to reorganize aggressively knowing they can reverse any mistake within 8 seconds.
