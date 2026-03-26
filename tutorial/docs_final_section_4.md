# The Mover System

The mover system in `src/mover.mjs` handles relocating and deleting Claude Code configuration items between scopes. Each movable category has a dedicated path resolver and move handler, while a central validation layer prevents illegal operations. The most notable implementation detail is how MCP server moves work -- since servers are entries inside JSON files rather than standalone files, moving them requires reading, modifying, and rewriting JSON on both sides.

## Category-to-Path Mapping

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

The pattern splits into two groups: categories stored under `~/.claude/` (memory, plan) use the encoded project directory for project-level paths, while categories stored in the repository itself (skill, rule, command, agent) use `<repoDir>/.claude/`. MCP servers are the outlier -- they live inside JSON files, not as standalone filesystem entries.

## Move Validation

`validateMove()` (lines 87-105) runs three checks before any move proceeds:

1. **Locked items cannot move** -- items with `locked: true` (configs, hooks, plugins) are rejected immediately.
2. **Same-scope moves are blocked** -- moving an item to the scope it already belongs to is a no-op error.
3. **Category whitelist** -- only `memory`, `skill`, `mcp`, `plan`, `command`, `agent`, and `rule` are movable. Sessions, configs, hooks, and plugins are excluded.

The central dispatcher `moveItem()` (lines 312-338) calls `validateMove()`, then routes to the appropriate category-specific handler.

## File-Based Moves

For most categories (memory, skill, plan, rule, command, agent), moving an item means physically relocating a file or directory from the source scope's path to the destination scope's path. The move handler:

1. Resolves the source and destination directories using the category's resolver.
2. Ensures the destination directory exists (creates it with `mkdir -p` semantics).
3. Calls `safeRename()` to move the file.

### `safeRename()` and the EXDEV Fallback (lines 21-33)

`safeRename()` wraps Node's `fs.rename()` with a critical fallback: if the rename throws an `EXDEV` error (meaning the source and destination are on different filesystems or mount points), it falls back to `cp()` followed by `rm()`. This matters in environments where `~/.claude/` and the project repository live on different volumes -- Docker containers, network mounts, or separate partitions.

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

The `isDir` parameter controls whether `cp` and `rm` operate recursively -- `true` when moving a skill directory, `false` when moving a single file like a memory or rule.

For skills specifically, the move relocates the entire skill directory (not just `SKILL.md`), preserving any additional files the skill contains.

## MCP Server JSON Moves (lines 249-300)

MCP server moves are structurally different because servers are key-value entries inside `.mcp.json` files, not standalone files. The move process:

1. **Read the source JSON file** and extract the server's config object by its name key.
2. **Read (or create) the destination JSON file** -- if the destination `.mcp.json` does not exist, start with `{ "mcpServers": {} }`.
3. **Check for name collision** -- if the destination already has a server with the same name, the move is rejected.
4. **Write the server config** into the destination's `mcpServers` object.
5. **Remove the server entry** from the source's `mcpServers` object.
6. **Write both files** sequentially (destination first, then source).

This ensures that at no point is the server config lost -- if the write to the destination succeeds but the source cleanup fails, the server exists in both locations rather than neither.

## Destination Filtering

`getValidDestinations()` (lines 418-444) determines which scopes an item can move to, applying category-specific rules:

- **memory, mcp, plan**: can move to any scope (global, workspace, or project).
- **skill, command, agent, rule**: can move to global or any scope that has a `repoDir` (since these categories store files inside the repository's `.claude/` directory, a scope without a repo directory has nowhere to put them).
- **Locked items**: always returns an empty array.

The current scope is always excluded from the results.

## Delete Operations (lines 340-412)

Each category has a dedicated delete function:

- **memory, plan, command, agent, rule**: calls `unlink()` to remove the single file.
- **skill**: calls `rm(path, { recursive: true })` to remove the entire skill directory and its contents.
- **mcp**: reads the JSON file, deletes the server key from the `mcpServers` object, and rewrites the file.
- **session**: deletes the `.jsonl` file and any subagent directory sharing the same UUID.

The dashboard's undo system (covered in Section 6) backs up file contents before deletion, enabling restoration via `POST /api/restore`.

## Why This Design Matters

The mover system's careful separation of file-based moves from JSON-manipulation moves reflects the reality that Claude Code's configuration is not uniform -- some items are files, others are entries inside shared JSON files. The `safeRename()` EXDEV fallback ensures moves work across filesystem boundaries without requiring users to think about mount points. And the validation layer prevents destructive operations on locked items, keeping configs, hooks, and plugins safe from accidental reorganization.
