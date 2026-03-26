# Overview & Architecture

Claude Code Organizer (CCO) gives you visibility into the scattered configuration files that Claude Code spreads across your filesystem -- memories, skills, MCP server configs, hooks, plans, rules, commands, agents, sessions, plugins, and configs -- and lets you move or delete them between scopes through a visual dashboard or an MCP server interface.

## The Problem: Scope Hierarchy Sprawl

Claude Code stores customizations at multiple levels: globally in `~/.claude/`, per-workspace for parent directories, and per-project under `~/.claude/projects/<encoded-path>/` or inside each repository's `.claude/` directory. A developer working across several repositories accumulates configuration at every level, with no built-in way to see what exists where or to reorganize items between scopes. CCO solves this by scanning every scope, building the parent-child hierarchy, and presenting everything in one unified view.

## Dual-Mode Design

CCO operates in two modes, selected at startup via a single CLI flag:

**Web Dashboard mode** (default): `node bin/cli.mjs` starts an HTTP server on port 3847 serving a three-panel browser UI. Developers interact through drag-and-drop to move items between scopes, search and filter across all categories, inspect file contents, and manage their context budget. The dashboard is designed for human exploration and bulk operations.

**MCP Server mode**: `node bin/cli.mjs --mcp` starts an MCP server over stdio transport. AI clients -- Claude Code itself, Cursor, Windsurf -- connect and call four tools programmatically: `scan_inventory`, `move_item`, `delete_item`, and `list_destinations`. This mode lets an AI assistant reorganize your Claude Code configuration on your behalf.

The entry point `bin/cli.mjs` (114 lines) handles mode routing: the `--mcp` flag sends execution to `src/mcp-server.mjs`, while the default path loads `src/server.mjs` for the HTTP dashboard. Before starting the dashboard, cli.mjs runs a pre-flight check (lines 20-34) verifying that `~/.claude/` exists and is readable, providing a clear error message rather than crashing with a confusing failure. It also auto-installs a `/cco` slash command skill to `~/.claude/skills/cco/SKILL.md` on first launch (lines 36-61), so users can invoke the dashboard directly from Claude Code.

## Zero-Build Architecture

CCO ships as pure ESM with no transpilation, no bundler, and no build step. The `package.json` declares `"type": "module"`, and every source file uses native ES module imports. The UI files in `src/ui/` -- `index.html`, `app.js`, and `style.css` -- are served directly by Node's built-in `http.createServer`. Client-side dependencies (SortableJS for drag-and-drop, Google Fonts for typography) load from CDN.

This means the entire runtime has a single production dependency: `@modelcontextprotocol/sdk` for the MCP server mode. An optional dependency, `ai-tokenizer`, provides accurate Claude-specific token counting; without it, CCO falls back to estimating tokens as UTF-8 byte length divided by 4 (`Math.ceil(Buffer.byteLength(text, "utf-8") / 4)`).

## Project Structure

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

`scanner.mjs` and `mover.mjs` are pure data modules with no HTTP or transport concerns -- they read the filesystem, build data structures, and return results. `server.mjs` and `mcp-server.mjs` are thin I/O wrappers that expose those modules over HTTP or stdio respectively. This separation means the scanning and moving logic is testable and reusable regardless of which interface is active.

## Why This Design Matters

The zero-build, single-dependency approach keeps CCO fast to install (`npx` just works) and easy to audit. Graceful pre-flight checks ensure that missing or inaccessible `~/.claude/` directories produce helpful messages instead of stack traces. The dual-mode architecture means the same scanning and moving logic serves both human developers browsing the dashboard and AI assistants operating programmatically -- no divergence, no duplication. And the pure-data/IO-layer separation ensures that the core logic remains decoupled from transport details, making the system straightforward to extend.
