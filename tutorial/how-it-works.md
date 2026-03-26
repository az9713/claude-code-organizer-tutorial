# How Claude Code Organizer Works

Claude Code Organizer (CCO) gives you visibility and control over the configuration files that Claude Code spreads across your filesystem. This document is a technical deep-dive into how it works internally — the algorithms, data structures, and design decisions behind the scanner, mover, context budget calculator, dashboard UI, and MCP server mode.

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Scope Discovery Engine](#2-scope-discovery-engine)
3. [The 11-Category Scanner](#3-the-11-category-scanner)
4. [The Mover System](#4-the-mover-system)
5. [Context Budget Calculator](#5-context-budget-calculator)
6. [The Dashboard UI](#6-the-dashboard-ui)
7. [MCP Server Mode](#7-mcp-server-mode)
8. [Plugin & Skill System](#8-plugin--skill-system)
9. [Technologies & Dependencies](#9-technologies--dependencies)

---

## 1. Overview & Architecture

Claude Code Organizer (CCO) gives you visibility into the scattered configuration files that Claude Code spreads across your filesystem — memories, skills, MCP server configs, hooks, plans, rules, commands, agents, sessions, plugins, and configs — and lets you move or delete them between scopes through a visual dashboard or an MCP server interface.

### The Problem: Scope Hierarchy Sprawl

Claude Code stores customizations at multiple levels: globally in `~/.claude/`, per-workspace for parent directories, and per-project under `~/.claude/projects/<encoded-path>/` or inside each repository's `.claude/` directory. A developer working across several repositories accumulates configuration at every level, with no built-in way to see what exists where or to reorganize items between scopes. CCO solves this by scanning every scope, building the parent-child hierarchy, and presenting everything in one unified view.

### Dual-Mode Design

CCO operates in two modes, selected at startup via a single CLI flag:

**Web Dashboard mode** (default): `node bin/cli.mjs` starts an HTTP server on port 3847 serving a three-panel browser UI. Developers interact through drag-and-drop to move items between scopes, search and filter across all categories, inspect file contents, and manage their context budget. The dashboard is designed for human exploration and bulk operations.

**MCP Server mode**: `node bin/cli.mjs --mcp` starts an MCP server over stdio transport. AI clients — Claude Code itself, Cursor, Windsurf — connect and call four tools programmatically: `scan_inventory`, `move_item`, `delete_item`, and `list_destinations`. This mode lets an AI assistant reorganize your Claude Code configuration on your behalf.

The entry point `bin/cli.mjs` (114 lines) handles mode routing: the `--mcp` flag sends execution to `src/mcp-server.mjs`, while the default path loads `src/server.mjs` for the HTTP dashboard. Before starting the dashboard, cli.mjs runs a pre-flight check (lines 20-34) verifying that `~/.claude/` exists and is readable, providing a clear error message rather than crashing with a confusing failure. It also auto-installs a `/cco` slash command skill to `~/.claude/skills/cco/SKILL.md` on first launch (lines 36-61), so users can invoke the dashboard directly from Claude Code.

### Zero-Build Architecture

CCO ships as pure ESM with no transpilation, no bundler, and no build step. The `package.json` declares `"type": "module"`, and every source file uses native ES module imports. The UI files in `src/ui/` — `index.html`, `app.js`, and `style.css` — are served directly by Node's built-in `http.createServer`. Client-side dependencies (SortableJS for drag-and-drop, Google Fonts for typography) load from CDN.

This means the entire runtime has a single production dependency: `@modelcontextprotocol/sdk` for the MCP server mode. An optional dependency, `ai-tokenizer`, provides accurate Claude-specific token counting; without it, CCO falls back to estimating tokens as UTF-8 byte length divided by 4 (`Math.ceil(Buffer.byteLength(text, "utf-8") / 4)`).

### Project Structure

The codebase separates concerns into pure data modules and I/O layers:

```
bin/cli.mjs           -- Entry point: mode routing, pre-flight checks, /cco skill auto-install
src/scanner.mjs       -- Pure data: scope discovery + 11-category item scanning (~1013 lines)
src/mover.mjs         -- Pure data: move + delete items between scopes (~445 lines)
src/server.mjs        -- I/O layer: HTTP server, REST API, context budget calculator (~689 lines)
src/mcp-server.mjs    -- I/O layer: MCP server wrapping scanner/mover as 4 tools (~135 lines)
src/tokenizer.mjs     -- Token counting with ai-tokenizer or UTF-8 byte/4 fallback (~59 lines)
src/ui/               -- Dashboard frontend: vanilla JS, no framework (~2,500 lines total)
.claude-plugin/       -- Plugin packaging for Claude Code's plugin registry
```

`scanner.mjs` and `mover.mjs` are pure data modules with no HTTP or transport concerns — they read the filesystem, build data structures, and return results. `server.mjs` and `mcp-server.mjs` are thin I/O wrappers that expose those modules over HTTP or stdio respectively. This separation means the scanning and moving logic is testable and reusable regardless of which interface is active.

### Why This Design Matters

The zero-build, single-dependency approach keeps CCO fast to install (`npx` just works) and easy to audit. Graceful pre-flight checks ensure that missing or inaccessible `~/.claude/` directories produce helpful messages instead of stack traces. The dual-mode architecture means the same scanning and moving logic serves both human developers browsing the dashboard and AI assistants operating programmatically — no divergence, no duplication. And the pure-data/IO-layer separation ensures that the core logic remains decoupled from transport details, making the system straightforward to extend.

---

## 2. Scope Discovery Engine

The scope discovery engine in `src/scanner.mjs` reconstructs Claude Code's configuration hierarchy by decoding the directory names under `~/.claude/projects/`, resolving them back to real filesystem paths, and building a tree of parent-child relationships. This is the foundation that every other part of CCO depends on — without knowing what scopes exist and how they relate, there is nothing to scan or move.

### How `~/.claude/projects/` Encodes Filesystem Paths

Claude Code creates a subdirectory under `~/.claude/projects/` for each project it has been used in. The directory name encodes the project's absolute filesystem path by replacing every `/` with `-` and prepending a leading `-`.

For example, the project at `/home/user/mycompany/repo1` becomes the directory `~/.claude/projects/-home-user-mycompany-repo1/`.

The problem: directory names on the filesystem can themselves contain dashes. A directory called `my-company` produces the same dash-separated encoding as two separate directories `my` and `company`. The encoded name `-home-user-my-company-repo1` is ambiguous — it could represent `/home/user/my-company/repo1` or `/home/user/my/company/repo1`. Resolving this ambiguity is what the greedy path decoding algorithm handles.

### The Greedy Path Decoding Algorithm

The function `resolveEncodedProjectPath()` (scanner.mjs lines 159-191) decodes an encoded directory name back to a real filesystem path using greedy longest-match-first resolution:

```javascript
async function resolveEncodedProjectPath(encoded) {
  const segments = encoded.replace(/^-/, "").split("-");
  let currentPath = "/";
  let i = 0;

  while (i < segments.length) {
    let matched = false;
    for (let end = segments.length; end > i; end--) {
      const candidate = segments.slice(i, end).join("-");
      const testPath = join(currentPath, candidate);
      if (await exists(testPath)) {
        const s = await safeStat(testPath);
        if (s && s.isDirectory()) {
          currentPath = testPath;
          i = end;
          matched = true;
          break;
        }
      }
    }
    if (!matched) {
      currentPath = join(currentPath, segments[i]);
      i++;
    }
  }

  if (await exists(currentPath)) return currentPath;
  return null;
}
```

#### Walkthrough

Given the encoded name `-home-user-my-company-repo1`, the algorithm splits on dashes to get segments `["home", "user", "my", "company", "repo1"]`, then resolves left to right:

1. At `/`, try the longest candidate first: `home-user-my-company-repo1` — does not exist. Shorten progressively until `home` matches `/home`. Advance to index 1.
2. At `/home`, try `user-my-company-repo1` — does not exist. Eventually `user` matches `/home/user`. Advance to index 2.
3. At `/home/user`, try `my-company-repo1` — does not exist. Try `my-company` — it exists as a real directory. Match. Advance to index 4.
4. At `/home/user/my-company`, try `repo1` — exists. Match. Done.
5. Result: `/home/user/my-company/repo1`.

The greedy strategy (longest match first) works because real directory names with dashes are typically longer than single path segments, so trying the longest candidate first resolves the ambiguity correctly in practice.

### Building the Scope Hierarchy

The `discoverScopes()` function (scanner.mjs lines 198-283) orchestrates the full discovery process:

1. **Create the global scope** — a singleton representing `~/.claude/` itself, always present.
2. **Read all directories** in `~/.claude/projects/` and call `resolveEncodedProjectPath()` on each.
3. **Filter empty scopes** — directories with no entries beyond `.DS_Store` are skipped (lines 235-244).
4. **Sort by path depth** (shorter paths first, then alphabetically) so that parent directories are processed before their children.
5. **Assign parent-child relationships** (lines 257-280): for each scope, find the deepest existing scope whose resolved `realPath` is a prefix of the current scope's path. That scope becomes the `parentId`. If no intermediate parent exists, the parent defaults to `global`.

Each scope object carries: `id` (the encoded directory name), `name` (human-readable), `type` (global, workspace, or project), `tag` (display label), `parentId`, `claudeProjectDir` (the `~/.claude/projects/<encoded>/` path), and `repoDir` (the resolved real filesystem path).

### Workspace Detection

A scope is classified as `workspace` rather than `project` when two conditions are met (lines 267-269):

1. Its parent is the `global` scope (it is a top-level entry).
2. At least one other scope's resolved path starts with this scope's path followed by `/`.

In other words, a workspace is a directory that contains child projects. For example, if both `/home/user/mycompany` and `/home/user/mycompany/repo1` appear as scopes, then `mycompany` is classified as a workspace. This distinction matters in the UI, where workspaces display as expandable tree nodes grouping their child projects.

### Why This Design Matters

The greedy path decoding algorithm is the critical piece — without it, CCO cannot map encoded directory names back to the filesystem and the entire scope hierarchy collapses. The approach is pragmatic: it uses filesystem existence checks rather than trying to maintain a separate mapping database, which means it works correctly even as projects are created, moved, or deleted outside of CCO. The parent-child relationship builder and workspace classifier then layer a meaningful hierarchy on top, giving both the dashboard UI and the MCP server a structured view of all configuration scopes.

---

## 3. The 11-Category Scanner

The scanner in `src/scanner.mjs` reads Claude Code's entire configuration surface by running 11 category-specific scanners in parallel for each discovered scope. Every scanner follows the same pattern: look in the right directories, parse the right file format, and return a normalized array of item objects. The main entry point, `scan()` (lines 979-1012), calls `discoverScopes()` first, then dispatches all category scanners concurrently per scope via `Promise.all`.

### Scanning Architecture

For each scope, the following scanners run in parallel (lines 986-997):

```
scanMemories, scanSkills, scanMcpServers, scanConfigs, scanHooks,
scanPlans, scanRules, scanCommands, scanAgents, scanSessions
```

Plugins are scanned separately at the global level only (line 1002) via `scanPlugins()`. Each scanner receives the scope object and returns an array of items with a common shape: `category`, `scopeId`, `name`, `fileName`, `description`, `path`, `size`, `mtime`, and category-specific fields.

### The 11 Categories

#### 1. Memories — `scanMemories()` (lines 325-364)

**Looks in:** `~/.claude/memory/*.md` (global) or `~/.claude/projects/<encoded>/memory/*.md` (project). The memory directory can be overridden by `settings.autoMemoryDirectory`, which resolves relative to `scope.repoDir` (line 333).

**Parses:** YAML frontmatter via `parseFrontmatter()` (lines 122-132), extracting `name`, `description`, and `type` (feedback, user, project, or reference). Files named `MEMORY.md` are excluded — that is the index file, not a memory item.

#### 2. Skills — `scanSkills()` (lines 366-452)

**Looks in:** `~/.claude/skills/` plus the platform-managed directory (global), or `<repoDir>/.claude/skills/` (project). Skips scanning when `repoDir` equals HOME to avoid double-counting via the `isGlobalClaudeDir()` guard (lines 28-30).

**Parses:** Reads `SKILL.md` in each subdirectory. Extracts the description from the first meaningful paragraph line after the heading — not from frontmatter. Also loads skill bundle information from `skills-lock.json` (v1) and `~/.agents/.skill-lock.json` (v3) via `loadSkillBundles()` (lines 295-321), recording the source URL if the skill was installed from a remote bundle.

**Special:** Supports symlinked skill directories (line 389), skips the `private` subdirectory (line 391), and counts total files and directory size for each skill.

#### 3. MCP Servers — `scanMcpServers()` (lines 454-598)

**Looks in (multi-source):** This is the most complex scanner because MCP server configs can live in five different locations at the global level:
1. `~/.claude/.mcp.json` (user scope via `claude mcp add -s user`)
2. `~/.mcp.json` (alternate user location)
3. `~/.claude.json` top-level `mcpServers` (default `claude mcp add`)
4. `/etc/claude-code/managed-mcp.json` (enterprise managed)
5. `mcpServers` inside `settings.json` / `settings.local.json`

For project scopes: `<repoDir>/.mcp.json`, project-scoped entries in `~/.claude.json` (at `projects[repoDir].mcpServers`), and `<repoDir>/.claude/settings.json`/`settings.local.json`.

**Parses:** JSON. Iterates the `mcpServers` object in each file, recording the server name, full config object (`mcpConfig`), and source file path. Scanning all sources surfaces duplicates, which the context budget calculator later deduplicates by name.

#### 4. Configs — `scanConfigs()` (lines 601-640)

**Looks in:** `CLAUDE.md` and `settings.json`/`settings.local.json` at both global and project levels, plus managed variants.

**Parses:** Only checks file existence and stats — does not parse content. All config items are marked `locked: true`, meaning they cannot be moved or deleted through the API. The dashboard instead generates Claude Code prompts for managing them.

#### 5. Hooks — `scanHooks()` (lines 642-689)

**Looks in:** `settings.json`, `settings.local.json`, and `managed-settings.json` at both levels.

**Parses:** JSON. Reads the `settings.hooks` object, where each key is an event name (like `PreToolUse`) containing an array of hook groups, each with a `hooks` array of individual commands. Extracts the event name, command or prompt text, and command type as `subType`. All hooks are `locked: true`.

#### 6. Plugins — `scanPlugins()` (lines 691-726)

**Looks in:** `~/.claude/plugins/cache/<org>/<plugin>/` — global scope only.

**Special:** Skips `temp_*` directories and hidden directories. Scans at the plugin name level, not version directories. All plugins are `locked: true`.

#### 7. Plans — `scanPlans()` (lines 728-773)

**Looks in:** `~/.claude/plans/` or a custom directory from `settings.plansDirectory` (global), or `~/.claude/projects/<encoded>/plans/` (project).

**Parses:** Markdown files. Extracts the first `# heading` as the description.

#### 8. Rules — `scanRules()` (lines 775-821)

**Looks in:** `~/.claude/rules/*.md` (global) or `<repoDir>/.claude/rules/*.md` (project).

**Parses:** Markdown files. Extracts the first `# heading` as the description.

#### 9. Commands — `scanCommands()` (lines 823-861)

**Looks in:** `~/.claude/commands/*.md` (global) or `<repoDir>/.claude/commands/*.md` (project).

**Parses:** YAML frontmatter for `name` and `description`.

#### 10. Agents — `scanAgents()` (lines 863-901)

**Looks in:** `~/.claude/agents/*.md` (global) or `<repoDir>/.claude/agents/*.md` (project).

**Parses:** YAML frontmatter for `name` and `description`.

#### 11. Sessions — `scanSessions()` (lines 903-967)

**Looks in:** `~/.claude/projects/<encoded>/*.jsonl` — project scopes only. Returns an empty array for the global scope.

**Parses:** JSONL files using **chunked I/O** for performance. Session files can be very large, so the scanner avoids reading them entirely:
- `readFirstLines(path, 10)` (lines 54-78): reads the first 10 lines via 8KB buffer chunks to find the `aiTitle` field near the top of the file.
- `readLastLines(path, 30, fileSize)` (lines 80-110): reads the last 30 lines via reverse 8KB chunks to find the last user message, skipping XML-like content and `tool_use` entries.

Sessions are `deletable: true` but not movable — they are inherently tied to the project scope where the conversation occurred.

### Why This Design Matters

Running all 11 scanners in parallel per scope keeps the full scan fast despite touching dozens of filesystem locations. The chunked I/O strategy for sessions is particularly important — session JSONL files can grow to megabytes, and reading only the head and tail extracts the title and last message without loading the entire file into memory. The multi-source MCP server scanning ensures nothing is hidden, surfacing duplicates that would otherwise silently shadow each other across configuration files.

---

## 4. The Mover System

The mover system in `src/mover.mjs` handles relocating and deleting Claude Code configuration items between scopes. Each movable category has a dedicated path resolver and move handler, while a central validation layer prevents illegal operations. The most notable implementation detail is how MCP server moves work — since servers are entries inside JSON files rather than standalone files, moving them requires reading, modifying, and rewriting JSON on both sides.

### Category-to-Path Mapping

Each movable category has a resolver function that determines where items live in a given scope:

| Category | Global Path | Project Path | Resolver |
|----------|-------------|--------------|----------|
| memory | `~/.claude/memory/` | `~/.claude/projects/<id>/memory/` | `resolveMemoryDir()` (line 40) |
| skill | `~/.claude/skills/` | `<repoDir>/.claude/skills/` | `resolveSkillDir()` (line 45) |
| plan | `~/.claude/plans/` | `~/.claude/projects/<id>/plans/` | `resolvePlanDir()` (line 52) |
| rule | `~/.claude/rules/` | `<repoDir>/.claude/rules/` | `resolveRuleDir()` (line 57) |
| command | `~/.claude/commands/` | `<repoDir>/.claude/commands/` | `resolveCommandDir()` (line 64) |
| agent | `~/.claude/agents/` | `<repoDir>/.claude/agents/` | `resolveAgentDir()` (line 71) |
| mcp | `~/.claude/.mcp.json` | `<repoDir>/.mcp.json` | `resolveMcpJson()` (line 78) |

The pattern splits into two groups: categories stored under `~/.claude/` (memory, plan) use the encoded project directory for project-level paths, while categories stored in the repository itself (skill, rule, command, agent) use `<repoDir>/.claude/`. MCP servers are the outlier — they live inside JSON files, not as standalone filesystem entries.

### Move Validation

`validateMove()` (lines 87-105) runs three checks before any move proceeds:

1. **Locked items cannot move** — items with `locked: true` (configs, hooks, plugins) are rejected immediately.
2. **Same-scope moves are blocked** — moving an item to the scope it already belongs to is a no-op error.
3. **Category whitelist** — only `memory`, `skill`, `mcp`, `plan`, `command`, `agent`, and `rule` are movable. Sessions, configs, hooks, and plugins are excluded.

The central dispatcher `moveItem()` (lines 312-338) calls `validateMove()`, then routes to the appropriate category-specific handler.

### File-Based Moves

For most categories (memory, skill, plan, rule, command, agent), moving an item means physically relocating a file or directory from the source scope's path to the destination scope's path. The move handler:

1. Resolves the source and destination directories using the category's resolver.
2. Ensures the destination directory exists (creates it with `mkdir -p` semantics).
3. Calls `safeRename()` to move the file.

#### `safeRename()` and the EXDEV Fallback (lines 21-33)

`safeRename()` wraps Node's `fs.rename()` with a critical fallback: if the rename throws an `EXDEV` error (meaning the source and destination are on different filesystems or mount points), it falls back to `cp()` followed by `rm()`. This matters in environments where `~/.claude/` and the project repository live on different volumes — Docker containers, network mounts, or separate partitions.

```javascript
async function safeRename(src, dest, isDir = false) {
  try {
    await rename(src, dest);
  } catch (err) {
    if (err.code === "EXDEV") {
      await cp(src, dest, { recursive: isDir });
      await rm(src, { recursive: isDir, force: true });
    } else {
      throw err;
    }
  }
}
```

The `isDir` parameter controls whether `cp` and `rm` operate recursively — `true` when moving a skill directory, `false` when moving a single file like a memory or rule.

For skills specifically, the move relocates the entire skill directory (not just `SKILL.md`), preserving any additional files the skill contains.

### MCP Server JSON Moves (lines 249-300)

MCP server moves are structurally different because servers are key-value entries inside `.mcp.json` files, not standalone files. The move process:

1. **Read the source JSON file** and extract the server's config object by its name key.
2. **Read (or create) the destination JSON file** — if the destination `.mcp.json` does not exist, start with `{ "mcpServers": {} }`.
3. **Check for name collision** — if the destination already has a server with the same name, the move is rejected.
4. **Write the server config** into the destination's `mcpServers` object.
5. **Remove the server entry** from the source's `mcpServers` object.
6. **Write both files** sequentially (destination first, then source).

This ensures that at no point is the server config lost — if the write to the destination succeeds but the source cleanup fails, the server exists in both locations rather than neither.

### Destination Filtering

`getValidDestinations()` (lines 418-444) determines which scopes an item can move to, applying category-specific rules:

- **memory, mcp, plan**: can move to any scope (global, workspace, or project).
- **skill, command, agent, rule**: can move to global or any scope that has a `repoDir` (since these categories store files inside the repository's `.claude/` directory, a scope without a repo directory has nowhere to put them).
- **Locked items**: always returns an empty array.

The current scope is always excluded from the results.

### Delete Operations (lines 340-412)

Each category has a dedicated delete function:

- **memory, plan, command, agent, rule**: calls `unlink()` to remove the single file.
- **skill**: calls `rm(path, { recursive: true })` to remove the entire skill directory and its contents.
- **mcp**: reads the JSON file, deletes the server key from the `mcpServers` object, and rewrites the file.
- **session**: deletes the `.jsonl` file and any subagent directory sharing the same UUID.

The dashboard's undo system backs up file contents before deletion, enabling restoration via `POST /api/restore`.

### Why This Design Matters

The mover system's careful separation of file-based moves from JSON-manipulation moves reflects the reality that Claude Code's configuration is not uniform — some items are files, others are entries inside shared JSON files. The `safeRename()` EXDEV fallback ensures moves work across filesystem boundaries without requiring users to think about mount points. And the validation layer prevents destructive operations on locked items, keeping configs, hooks, and plugins safe from accidental reorganization.

---

## 5. Context Budget Calculator

The context budget calculator in `src/server.mjs` answers a practical question: how many tokens does Claude Code consume with your current configuration before you type a single word? Every CLAUDE.md file, every rule, every skill description, every MCP tool schema takes up space in the context window. The budget calculator tokenizes all of it, walks the scope inheritance chain, and returns a detailed breakdown so developers can see exactly where their context is going.

### What Gets Counted

The calculator splits items into two categories based on how Claude Code loads them.

#### Always-Loaded Items (lines 156-176)

These are present in every conversation, unconditionally:

- **System prompt**: ~6,500 tokens (estimated constant)
- **System tools (loaded part)**: ~6,000 tokens (estimated constant)
- **CLAUDE.md files**: full content from `CLAUDE.md`, `.claude/CLAUDE.md`, and managed `CLAUDE.md` — read and tokenized
- **Rules** (`.claude/rules/*.md`): all rule files, full content
- **MEMORY.md**: first 200 lines or 25KB, whichever comes first (lines 267-268)
- **Skill descriptions**: the name and frontmatter description of each skill (not the full SKILL.md body)
- **Commands and agents**: full file content of all `.md` files in commands and agents directories

The always-loaded categories are: `skill`, `rule`, `command`, and `agent` (line 176). Config items included are specifically `CLAUDE.md`, `.claude/CLAUDE.md`, and `CLAUDE.md (managed)` (line 178).

#### Deferred Items (lines 165-173)

These are reserved in the context window but only loaded on demand (typically via ToolSearch):

- **System tools (deferred part)**: ~10,500 tokens (estimated constant)
- **MCP tool definitions**: ~3,100 tokens per unique MCP server (line 308)

For MCP servers, Claude Code deduplicates by name with priority order: local > project > user. The budget calculator counts unique server names, not total entries across all config files, because duplicates shadow each other and only one instance consumes context.

### Scope Chain Inheritance (lines 146-153)

The calculator does not just count items in the requested scope — it walks the full parent chain by following `parentId` links up to the global scope. For example, requesting the budget for a project scope collects:

1. Items from the project scope itself (current scope)
2. Items from the workspace scope, if any (inherited)
3. Items from the global scope (inherited)

Current-scope and inherited items are tokenized separately, so the API response distinguishes between "your project's own budget" and "what you inherit from parent scopes."

### Token Counting (tokenizer.mjs)

The `countTokens()` function (tokenizer.mjs lines 45-49) uses lazy initialization (lines 19-38): on first call, it attempts to import `ai-tokenizer` and `ai-tokenizer/encoding` with Claude's encoding. If the optional dependency is not installed, it falls back to:

```javascript
Math.ceil(Buffer.byteLength(text, "utf-8") / 4)
```

This estimation is roughly 75-85% accurate compared to the real tokenizer. The function returns `{ tokens, confidence }` where confidence is either `"measured"` (real tokenizer) or `"estimated"` (byte fallback).

### System Overhead Constants (lines 293-296)

The calculator includes fixed overhead that comes from Claude Code's own system setup:

```javascript
const SYSTEM_LOADED = 12500;   // system prompt (~6.5K) + loaded system tools (~6K)
const SYSTEM_DEFERRED = 10500; // deferred system tools
// MCP: mcpUniqueCount * 3100 tokens per unique server
```

These constants are hardcoded estimates based on measured values of Claude Code's system prompt and tool definitions.

### API Endpoint

`GET /api/context-budget?scope=<id>&limit=<number>` (lines 136-356) returns a JSON response with this structure:

```json
{
  "alwaysLoaded": {
    "currentScope": { "items": [...], "total": 1250 },
    "inherited": { "items": [...], "total": 8400 },
    "system": 12500,
    "total": 22150
  },
  "deferred": {
    "currentScope": { "items": [], "total": 0 },
    "inherited": { "items": [], "total": 0 },
    "systemTools": 10500,
    "mcpToolSchemas": 9300,
    "mcpServerCount": 5,
    "mcpUniqueCount": 3,
    "total": 19800
  },
  "total": 41950,
  "contextLimit": 200000,
  "percentUsed": 11.1,
  "percentWithDeferred": 21.0,
  "method": "estimated"
}
```

The `contextLimit` defaults to 200,000 tokens. The `limit` query parameter allows overriding this (the dashboard UI provides a toggle for 200K vs 1M context windows). The `method` field reflects whether token counts came from the real tokenizer or the byte estimation fallback.

### Why This Design Matters

Context budget visibility solves a real developer pain point: mysterious "context too long" errors or degraded Claude Code performance caused by accumulated configuration. By tokenizing everything before the conversation starts and presenting the breakdown by scope and category, the calculator lets developers identify which CLAUDE.md file, which set of rules, or which MCP server fleet is consuming the most budget — and then use the mover system to reorganize items to the scopes where they are actually needed.

---

## 6. The Dashboard UI

The dashboard is a three-panel browser interface built entirely in vanilla JavaScript — no framework, no build step, no bundler. The frontend consists of `src/ui/index.html` (layout and modals), `src/ui/app.js` (~2,245 lines of application logic), and `src/ui/style.css` (theming with oklch colors). It communicates with the Node.js backend via a REST API defined in `src/server.mjs`, and uses SortableJS loaded from CDN for drag-and-drop.

### Three-Panel Layout

The layout divides the viewport into three resizable panels separated by draggable dividers:

1. **Left Sidebar** (`#sidebar`): A scope tree with expandable nodes showing category counts per scope. Contains the search input, collapse-all button, theme toggle, export button, and footer links. During drag operations, the sidebar collapses to a compact scope-tree-only view to serve as a drop target.

2. **Main Content** (`#mainContent`): Displays the selected scope's items. The content header shows the scope name, inheritance chain, and a context budget button. A filter bar with category pills lets users toggle visibility by category. Items are grouped by category with sort controls (name, size, date).

3. **Right Detail Panel** (`#detailPanel`): Shows metadata for the selected item — scope, type, description, size, dates, and file path. Includes a file preview, "Claude Code Prompt" action buttons, and move/delete/open controls. This panel alternates with the **Context Budget Panel** (`#ctxBudgetPanel`), which displays token usage as a visual bar chart.

The panels are separated by draggable resizers (`#resizerLeft`, `#resizerRight`) implemented in `setupResizer()` (app.js lines 397-438), allowing users to adjust panel widths.

### REST API Routes

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

### SortableJS Drag-and-Drop

**Initialization:** SortableJS v1.15.6 is loaded from `cdn.jsdelivr.net`. `initSortable()` (app.js lines 1310-1378) creates Sortable instances on all `.sortable-zone` elements. Each zone has a `data-group` attribute matching a category group (memory, skill, mcp, etc.), and items with the same group can be dragged between zones.

**Drag flow:**
1. `onStart`: stores the dragged item, adds a `drag-active` class to the sidebar, and collapses it to scope-tree-only view.
2. The user drags the item to a different scope's sortable zone or drops it on a sidebar scope block.
3. `onEnd`: if the source and destination zones differ, shows the drag confirm modal via `showDragConfirm()`.
4. On confirmation, `doMove()` calls `POST /api/move`.
5. After the move completes, `refreshUI()` re-scans and re-renders everything.

**Sidebar drop zones** (lines 1380-1444): separate `dragover`/`drop` handlers on the document allow dropping items directly onto sidebar scope blocks, not just sortable zones. A `.drop-target` CSS class provides visual highlighting during hover.

**Locked item behavior**: when a locked item (config, hook, plugin) is dragged to a scope block, instead of calling the move API, the UI generates a Claude Code prompt and copies it to the clipboard (lines 1413-1421). This lets users paste the prompt into Claude Code to perform the operation manually.

### Search and Filter

**Search** (app.js lines 145-151): filters items by matching the query against a concatenation of `[name, description, category, subType, path].join(" ")` (lines 2143-2153). The search also filters sidebar scope visibility, hiding scopes with no matching items.

**Category filters** (app.js lines 196-221): toggle pills in the filter bar. An `activeFilters` Set tracks which categories are visible. Items must match at least one active filter; when no filters are active, all items are shown.

### Undo System

The dashboard supports undoing both moves and deletes:

**Move undo** (app.js lines 1835-1858): after a successful move, the UI creates an `undoFn` that calls `POST /api/move` in reverse, moving the item back to its original scope. A toast notification with an "Undo" button appears for 8 seconds.

**Delete undo** (app.js lines 1869-1952): before deleting, the UI reads the file content as a `backupContent` string. For MCP items, it stores `{ name, config, mcpJsonPath }` as `mcpBackup`. After deletion, the `undoFn` calls either `POST /api/restore` (for files) or `POST /api/restore-mcp` (for MCP entries). For skills (directories), restoration creates the directory and writes `SKILL.md` from the backup.

**Toast notifications** (app.js lines 1961-1983): appear at the bottom of the viewport. Support error styling, undo buttons, persistent close buttons, and auto-hide (4 seconds default, 8 seconds when an undo action is available).

### Additional Features

**Bulk operations** (lines 311-357): a checkbox select mode enables multi-select. Bulk move (restricted to items of the same category) and bulk delete operate on the selection.

**Claude Code Prompt buttons** (`renderCcActions()`, lines 811-896): category-specific prompts that users copy and paste into Claude Code. Actions include "Explain This", "Edit", "Resume Session", "Modify", and "Remove". Each prompt is multi-step, asking Claude to read the item, explain it, and confirm before acting.

**Theme toggle**: switches a `.dark` class on the document body (line 364). CSS uses the `oklch` color space with separate light and dark variable sets for consistent color rendering.

**Export** (lines 913-939): calls `POST /api/export`, which copies all scanned items to `~/.claude/exports/cco-backup-<timestamp>/` organized by `scopeId/category/`.

### Why This Design Matters

The vanilla JS architecture with zero build tooling means the dashboard loads instantly from the Node server with no compilation step — the same files developers can read in their editor are exactly what runs in the browser. SortableJS provides production-grade drag-and-drop without framework coupling. The undo system, which backs up content before destructive operations, gives users confidence to reorganize aggressively knowing they can reverse any mistake within 8 seconds.

---

## 7. MCP Server Mode

When launched with `--mcp`, CCO becomes an MCP server that AI clients can connect to over stdio transport. The implementation in `src/mcp-server.mjs` (135 lines) wraps the same `scanner.mjs` and `mover.mjs` modules that power the dashboard, exposing them as four tools with Zod-validated input schemas. This means an AI assistant like Claude Code can scan, reorganize, and clean up its own configuration programmatically.

### Transport

The server uses `StdioServerTransport` from `@modelcontextprotocol/sdk` (line 133-134), communicating over stdin/stdout. AI clients configure the connection in their MCP settings — for example, adding an entry to `.mcp.json` that runs `npx @mcpware/claude-code-organizer --mcp`.

### The 4 Tools

#### 1. `scan_inventory` (lines 41-51)

**Parameters:** None.

**Behavior:** Calls `freshScan()`, which invokes `scan()` from `scanner.mjs` to discover all scopes and scan every category. Returns the complete scan result as JSON — scopes, items, and counts.

**Use case:** An AI assistant calls this first to understand what configuration exists across all scopes before deciding what to move or delete.

#### 2. `move_item` (lines 53-79)

**Parameters (Zod-validated):**
- `category`: enum of `['memory', 'skill', 'mcp', 'plan', 'session', 'command', 'agent', 'rule']`
- `name`: string — the item's name
- `fromScopeId`: string — source scope ID
- `toScopeId`: string — destination scope ID

**Behavior:** If no cached scan data exists, calls `freshScan()` first. Locates the item via `findItem()` (lines 32-39), which searches the cached data by `category`, `name` (or `fileName`), and `scopeId`. Then calls `moveItem()` from `mover.mjs`, which runs validation and the category-specific move handler. After a successful move, refreshes the cache via `freshScan()`.

#### 3. `delete_item` (lines 81-106)

**Parameters (Zod-validated):**
- `category`: enum (same as `move_item`)
- `name`: string
- `scopeId`: string

**Behavior:** Same pattern as `move_item` — ensures cached data exists, finds the item, calls `deleteItem()` from `mover.mjs`, and refreshes the cache on success.

#### 4. `list_destinations` (lines 108-131)

**Parameters (Zod-validated):**
- `category`: enum (same as above)
- `name`: string
- `scopeId`: string

**Returns:** An array of valid destination scopes where the item can be moved, as determined by `getValidDestinations()` from `mover.mjs`. This lets the AI client present options or make an informed decision before calling `move_item`.

### Item Lookup

`findItem()` (lines 32-39) searches the cached scan data for an item matching the given `category`, `scopeId`, and either `name` or `fileName`. This dual-field matching handles the case where an item's display name differs from its filename (for example, a memory file named `user_role.md` with a frontmatter `name` of "User Role").

### Caching Strategy

A module-level `cachedData` variable (line 21) stores the most recent scan result. The cache is populated or refreshed by `freshScan()`, which is called:
- On every `scan_inventory` call (always fresh)
- Before `move_item`, `delete_item`, or `list_destinations` if the cache is empty
- After every successful `move_item` or `delete_item` (to reflect the changed state)

This means the first tool call in a session always triggers a full scan, and the cache stays consistent with the filesystem after mutations.

### Server Setup

The MCP server is created using `McpServer` from the SDK (line 7), registered with the name `"claude-code-organizer"` and a hardcoded version `"0.5.0"` (maintained separately from the npm package version). Each tool is registered with a description, a Zod schema for input validation, and an async handler function. The server connects to the transport on startup:

```javascript
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Why This Design Matters

The MCP server mode transforms CCO from a human-only tool into one that AI assistants can operate autonomously. Because it wraps the same scanner and mover modules as the dashboard, there is no behavioral divergence between what a human sees in the browser and what an AI client can do via tools. The Zod validation ensures that AI clients cannot pass malformed input, and the automatic cache refresh after mutations means the AI always works with an accurate view of the filesystem state. The thin 135-line implementation demonstrates the value of CCO's pure-data/IO-layer separation — adding a new transport required only writing tool definitions, not reimplementing any scanning or moving logic.

---

## 8. Plugin & Skill System

CCO integrates with Claude Code through two complementary mechanisms: a plugin package for the Claude Code plugin registry, and a standalone skill that auto-installs on first dashboard launch. Both approaches give users a way to invoke CCO from within Claude Code, but they serve different distribution paths — the plugin bundles everything for registry-based installation, while the auto-installed skill provides a lightweight shortcut for `npx` users.

### `.claude-plugin/` Package Structure

The `.claude-plugin/` directory at the project root makes CCO installable as a Claude Code plugin. It contains three files:

#### `plugin.json` — Plugin Metadata

Declares the plugin's identity for the registry:

```json
{
  "name": "claude-code-organizer",
  "version": "...",
  "description": "...",
  "author": "...",
  "homepage": "...",
  "keywords": [...]
}
```

This is read by Claude Code's plugin system when browsing or installing plugins.

#### `settings.json` — Auto-Configuration

When the plugin is installed, this file automatically configures two things:

1. **MCP server registration**: adds a `claude-code-organizer` entry to the user's MCP server config, configured to run via `npx @mcpware/claude-code-organizer --mcp`.
2. **Bash permission**: grants `Bash(npx @mcpware/claude-code-organizer*)` so Claude Code can launch the dashboard without prompting for permission.

This means installing the plugin immediately enables both MCP server mode and dashboard access without manual configuration.

#### `skills/organize.md` — Plugin-Bundled Skill

A skill definition bundled with the plugin. Uses frontmatter to declare its interface:

```yaml
---
name: organize
description: Open the Claude Code Organizer dashboard to view and manage memories, skills, MCP servers, and hooks across all scopes.
argument-hint: "[port]"
---
```

The body contains instructions for Claude Code to run the dashboard via `npx`. Note that the plugin-bundled skill omits the `allowed-tools` frontmatter field — tool permissions are managed by the plugin's `settings.json` instead.

### Auto-Install of `/cco` Skill (cli.mjs lines 36-61)

Independent of the plugin system, CCO auto-installs a `/cco` slash command skill when the dashboard starts for the first time. This ensures that even users who install via `npx` (without the plugin) get a convenient way to relaunch the dashboard from within Claude Code.

The auto-install logic in `bin/cli.mjs`:

1. Checks if `~/.claude/skills/cco/SKILL.md` already exists.
2. If not, creates the `~/.claude/skills/cco/` directory.
3. Writes `SKILL.md` with the skill definition:
   ```yaml
   ---
   name: cco
   description: Open Claude Code Organizer dashboard to manage memories, skills, MCP servers across scopes
   ---
   ```
   The body instructs Claude Code to run `npx @mcpware/claude-code-organizer` to open the dashboard at `localhost:3847`.
4. Prints a success message: "Installed /cco skill globally".
5. Silently catches and ignores write permission errors — if the user's filesystem prevents the write, the dashboard still starts normally.

This runs only in dashboard mode (not MCP mode) and only on the first launch. Subsequent launches detect the existing file and skip the install.

### Standalone Skill Definition

The `skills/organize/SKILL.md` file in the repository is the standalone skill definition, which can be installed manually or via the auto-install mechanism. Its frontmatter is more complete than the plugin-bundled version:

```yaml
---
name: organize
description: Open the Claude Code Organizer dashboard...
argument-hint: [--port <number>]
allowed-tools: [Bash, Read]
---
```

The `allowed-tools` field declares that Claude Code needs `Bash` and `Read` tool access to execute this skill. The `argument-hint` field tells Claude Code what arguments the skill accepts, displayed when the user types `/organize`.

### Skill Definition Format

Claude Code skills follow a consistent format:

- **Frontmatter** (YAML between `---` delimiters): declares `name`, `description`, and optional fields like `argument-hint` and `allowed-tools`.
- **Body** (markdown after the frontmatter): contains the instructions that Claude Code follows when the skill is invoked. For CCO's skills, this instructs Claude Code to run `npx @mcpware/claude-code-organizer` with appropriate arguments.

Skills are discovered by scanning `SKILL.md` files in subdirectories of `~/.claude/skills/` (global) or `<repoDir>/.claude/skills/` (project-scoped). The scanner extracts the description from the first meaningful paragraph line after the heading, not from the frontmatter — a detail covered in Section 3.

### Why This Design Matters

The dual distribution strategy — plugin registry and auto-installed skill — ensures CCO is accessible regardless of how a user discovers it. Plugin users get automatic MCP server setup and bash permissions. `npx` users get a `/cco` slash command on first launch. Both paths converge on the same functionality. The auto-install's graceful failure mode (silently catching permission errors) means the dashboard never fails to start just because skill installation was blocked — it treats the skill as a convenience, not a dependency.

---

## 9. Technologies & Dependencies

CCO is built on a deliberately minimal technology stack: Node.js 20+ with pure ESM, a single production dependency, and zero build tooling. This section catalogs every runtime requirement, dependency, and infrastructure component.

### Runtime Requirements

- **Node.js >= 20**: declared in `package.json` as `"engines": { "node": ">=20" }`. Required for native ES module support and modern `fs/promises` APIs.
- **No OS restriction**: works on macOS, Linux, and Windows. Platform-specific paths are handled in `scanner.mjs` (lines 17-20): macOS uses `/Library/Application Support/ClaudeCode/` for managed configuration, Linux uses `/etc/claude-code/`.

### Production Dependencies

CCO has exactly one production dependency:

- **`@modelcontextprotocol/sdk` ^1.27.1**: the MCP server framework. Provides `McpServer` for tool registration, `StdioServerTransport` for stdio communication, and re-exports `zod` for input schema validation. Used only in `src/mcp-server.mjs` — the dashboard mode does not import it at startup.

### Optional Dependency

- **`ai-tokenizer` ^1.0.6**: a Claude-specific tokenizer for accurate token counting. Declared in `"optionalDependencies"` in `package.json`, so `npm install` will not fail if it cannot be installed. When available, the context budget calculator uses it for measured token counts (confidence: `"measured"`). When absent, `src/tokenizer.mjs` falls back to `Math.ceil(Buffer.byteLength(text, "utf-8") / 4)` (confidence: `"estimated"`), which is roughly 75-85% accurate.

### Dev Dependencies

- **`@playwright/test` ^1.58.2**: end-to-end browser testing framework. Tests launch the dashboard, interact with the UI, and verify that scanning, moving, and deleting work through the full stack.

### CDN-Loaded Client Libraries

The dashboard frontend loads two libraries from CDN, avoiding any need for `node_modules` on the client side:

- **SortableJS 1.15.6**: `https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js` — provides drag-and-drop functionality for moving items between scope zones.
- **Google Fonts**: Lato (weights 400, 700, 900) for UI text and Geist Mono (weights 400, 500) for code and monospace elements.

### HTTP Server

The dashboard runs on Node's built-in `http.createServer` — no Express, Koa, or other HTTP framework. Key details:

- **Default port**: 3847
- **Auto-retry**: if the default port is in use, the server tries up to 10 consecutive ports (server.mjs lines 650-687)
- **Auto-open**: on macOS (`open`) and Linux (`xdg-open`), the browser opens automatically on startup (cli.mjs lines 108-113)
- **Non-blocking update check**: on startup, queries the npm registry to check for newer versions without blocking the server from accepting requests

### Zero-Build Architecture

The project uses no transpilation, no bundler, and no build step:

- `package.json` declares `"type": "module"` for native ESM
- All source files use `import`/`export` syntax directly
- `src/ui/` files (HTML, CSS, JS) are served as static files by the Node server
- CSS uses modern features (`oklch` color space, CSS custom properties) with separate light/dark theme variable sets
- No TypeScript, no JSX, no preprocessors

### CI/CD Pipeline

The GitHub Actions workflow `.github/workflows/publish.yml` triggers on push of `v*` tags and runs a single job:

1. **Checkout** via `actions/checkout@v5`
2. **Setup Node.js** with LTS version and npm registry configuration
3. **Install dependencies** with `npm ci` or `npm install --ignore-scripts`
4. **Dedup check**: queries the npm registry to see if the version is already published, skipping the publish if so
5. **Publish to npm**: `npm publish --access public` using an `NPM_TOKEN` secret, with provenance enabled (`"publishConfig": { "provenance": true }` in `package.json`)
6. **Create GitHub Release**: via `softprops/action-gh-release@v2` with auto-generated release notes
7. **Install mcp-publisher**: downloads the MCP Registry publisher binary
8. **OIDC authentication**: authenticates to the MCP Registry via GitHub Actions OIDC
9. **Publish to MCP Registry**: registers or updates the MCP server entry (continues on error to avoid blocking the release)

#### Required Permissions

The workflow requires two GitHub Actions permissions:
- `id-token: write` — for MCP Registry OIDC authentication
- `contents: write` — for creating GitHub Releases

### Why This Design Matters

The single production dependency and zero-build architecture mean CCO installs in seconds via `npx` with no compilation delays or platform-specific build failures. The optional tokenizer dependency degrades gracefully — users who do not install it still get useful (if approximate) context budget calculations. CDN-loaded client libraries keep the npm package small while providing production-quality UI components. And the CI/CD pipeline publishes to three distribution channels (npm, GitHub Releases, MCP Registry) in a single automated workflow, ensuring that every tagged release is immediately available everywhere users might look for it.

---

*Generated by a 3-agent team (researcher + writer + editor) using the claude-code-organizer agent team workflow — March 2026*
