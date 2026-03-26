# MCP Server Mode

When launched with `--mcp`, CCO becomes an MCP server that AI clients can connect to over stdio transport. The implementation in `src/mcp-server.mjs` (135 lines) wraps the same `scanner.mjs` and `mover.mjs` modules that power the dashboard, exposing them as four tools with Zod-validated input schemas. This means an AI assistant like Claude Code can scan, reorganize, and clean up its own configuration programmatically.

## Transport

The server uses `StdioServerTransport` from `@modelcontextprotocol/sdk` (line 133-134), communicating over stdin/stdout. AI clients configure the connection in their MCP settings -- for example, adding an entry to `.mcp.json` that runs `npx @mcpware/claude-code-organizer --mcp`.

## The 4 Tools

### 1. `scan_inventory` (lines 41-51)

**Parameters:** None.

**Behavior:** Calls `freshScan()`, which invokes `scan()` from `scanner.mjs` to discover all scopes and scan every category. Returns the complete scan result as JSON -- scopes, items, and counts.

**Use case:** An AI assistant calls this first to understand what configuration exists across all scopes before deciding what to move or delete.

### 2. `move_item` (lines 53-79)

**Parameters (Zod-validated):**
- `category`: enum of `['memory', 'skill', 'mcp', 'plan', 'session', 'command', 'agent', 'rule']`
- `name`: string -- the item's name
- `fromScopeId`: string -- source scope ID
- `toScopeId`: string -- destination scope ID

**Behavior:** If no cached scan data exists, calls `freshScan()` first. Locates the item via `findItem()` (lines 32-39), which searches the cached data by `category`, `name` (or `fileName`), and `scopeId`. Then calls `moveItem()` from `mover.mjs`, which runs validation and the category-specific move handler. After a successful move, refreshes the cache via `freshScan()`.

### 3. `delete_item` (lines 81-106)

**Parameters (Zod-validated):**
- `category`: enum (same as `move_item`)
- `name`: string
- `scopeId`: string

**Behavior:** Same pattern as `move_item` -- ensures cached data exists, finds the item, calls `deleteItem()` from `mover.mjs`, and refreshes the cache on success.

### 4. `list_destinations` (lines 108-131)

**Parameters (Zod-validated):**
- `category`: enum (same as above)
- `name`: string
- `scopeId`: string

**Returns:** An array of valid destination scopes where the item can be moved, as determined by `getValidDestinations()` from `mover.mjs`. This lets the AI client present options or make an informed decision before calling `move_item`.

## Item Lookup

`findItem()` (lines 32-39) searches the cached scan data for an item matching the given `category`, `scopeId`, and either `name` or `fileName`. This dual-field matching handles the case where an item's display name differs from its filename (for example, a memory file named `user_role.md` with a frontmatter `name` of "User Role").

## Caching Strategy

A module-level `cachedData` variable (line 21) stores the most recent scan result. The cache is populated or refreshed by `freshScan()`, which is called:
- On every `scan_inventory` call (always fresh)
- Before `move_item`, `delete_item`, or `list_destinations` if the cache is empty
- After every successful `move_item` or `delete_item` (to reflect the changed state)

This means the first tool call in a session always triggers a full scan, and the cache stays consistent with the filesystem after mutations.

## Server Setup

The MCP server is created using `McpServer` from the SDK (line 7), registered with the name `"claude-code-organizer"` and a hardcoded version `"0.5.0"` (maintained separately from the npm package version). Each tool is registered with a description, a Zod schema for input validation, and an async handler function. The server connects to the transport on startup:

```javascript
const transport = new StdioServerTransport();
await server.connect(transport);
```

## Why This Design Matters

The MCP server mode transforms CCO from a human-only tool into one that AI assistants can operate autonomously. Because it wraps the same scanner and mover modules as the dashboard, there is no behavioral divergence between what a human sees in the browser and what an AI client can do via tools. The Zod validation ensures that AI clients cannot pass malformed input, and the automatic cache refresh after mutations means the AI always works with an accurate view of the filesystem state. The thin 135-line implementation demonstrates the value of CCO's pure-data/IO-layer separation -- adding a new transport required only writing tool definitions, not reimplementing any scanning or moving logic.
