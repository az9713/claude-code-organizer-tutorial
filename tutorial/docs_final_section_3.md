# The 11-Category Scanner

The scanner in `src/scanner.mjs` reads Claude Code's entire configuration surface by running 11 category-specific scanners in parallel for each discovered scope. Every scanner follows the same pattern: look in the right directories, parse the right file format, and return a normalized array of item objects. The main entry point, `scan()` (lines 979-1012), calls `discoverScopes()` first, then dispatches all category scanners concurrently per scope via `Promise.all`.

## Scanning Architecture

For each scope, the following scanners run in parallel (lines 986-997):

```
scanMemories, scanSkills, scanMcpServers, scanConfigs, scanHooks,
scanPlans, scanRules, scanCommands, scanAgents, scanSessions
```

Plugins are scanned separately at the global level only (line 1002) via `scanPlugins()`. Each scanner receives the scope object and returns an array of items with a common shape: `category`, `scopeId`, `name`, `fileName`, `description`, `path`, `size`, `mtime`, and category-specific fields.

## The 11 Categories

### 1. Memories -- `scanMemories()` (lines 325-364)

**Looks in:** `~/.claude/memory/*.md` (global) or `~/.claude/projects/<encoded>/memory/*.md` (project). The memory directory can be overridden by `settings.autoMemoryDirectory`, which resolves relative to `scope.repoDir` (line 333).

**Parses:** YAML frontmatter via `parseFrontmatter()` (lines 122-132), extracting `name`, `description`, and `type` (feedback, user, project, or reference). Files named `MEMORY.md` are excluded -- that is the index file, not a memory item.

### 2. Skills -- `scanSkills()` (lines 366-452)

**Looks in:** `~/.claude/skills/` plus the platform-managed directory (global), or `<repoDir>/.claude/skills/` (project). Skips scanning when `repoDir` equals HOME to avoid double-counting via the `isGlobalClaudeDir()` guard (lines 28-30).

**Parses:** Reads `SKILL.md` in each subdirectory. Extracts the description from the first meaningful paragraph line after the heading -- not from frontmatter. Also loads skill bundle information from `skills-lock.json` (v1) and `~/.agents/.skill-lock.json` (v3) via `loadSkillBundles()` (lines 295-321), recording the source URL if the skill was installed from a remote bundle.

**Special:** Supports symlinked skill directories (line 389), skips the `private` subdirectory (line 391), and counts total files and directory size for each skill.

### 3. MCP Servers -- `scanMcpServers()` (lines 454-598)

**Looks in (multi-source):** This is the most complex scanner because MCP server configs can live in five different locations at the global level:
1. `~/.claude/.mcp.json` (user scope via `claude mcp add -s user`)
2. `~/.mcp.json` (alternate user location)
3. `~/.claude.json` top-level `mcpServers` (default `claude mcp add`)
4. `/etc/claude-code/managed-mcp.json` (enterprise managed)
5. `mcpServers` inside `settings.json` / `settings.local.json`

For project scopes: `<repoDir>/.mcp.json`, project-scoped entries in `~/.claude.json` (at `projects[repoDir].mcpServers`), and `<repoDir>/.claude/settings.json`/`settings.local.json`.

**Parses:** JSON. Iterates the `mcpServers` object in each file, recording the server name, full config object (`mcpConfig`), and source file path. Scanning all sources surfaces duplicates, which the context budget calculator later deduplicates by name.

### 4. Configs -- `scanConfigs()` (lines 601-640)

**Looks in:** `CLAUDE.md` and `settings.json`/`settings.local.json` at both global and project levels, plus managed variants.

**Parses:** Only checks file existence and stats -- does not parse content. All config items are marked `locked: true`, meaning they cannot be moved or deleted through the API. The dashboard instead generates Claude Code prompts for managing them.

### 5. Hooks -- `scanHooks()` (lines 642-689)

**Looks in:** `settings.json`, `settings.local.json`, and `managed-settings.json` at both levels.

**Parses:** JSON. Reads the `settings.hooks` object, where each key is an event name (like `PreToolUse`) containing an array of hook groups, each with a `hooks` array of individual commands. Extracts the event name, command or prompt text, and command type as `subType`. All hooks are `locked: true`.

### 6. Plugins -- `scanPlugins()` (lines 691-726)

**Looks in:** `~/.claude/plugins/cache/<org>/<plugin>/` -- global scope only.

**Special:** Skips `temp_*` directories and hidden directories. Scans at the plugin name level, not version directories. All plugins are `locked: true`.

### 7. Plans -- `scanPlans()` (lines 728-773)

**Looks in:** `~/.claude/plans/` or a custom directory from `settings.plansDirectory` (global), or `~/.claude/projects/<encoded>/plans/` (project).

**Parses:** Markdown files. Extracts the first `# heading` as the description.

### 8. Rules -- `scanRules()` (lines 775-821)

**Looks in:** `~/.claude/rules/*.md` (global) or `<repoDir>/.claude/rules/*.md` (project).

**Parses:** Markdown files. Extracts the first `# heading` as the description.

### 9. Commands -- `scanCommands()` (lines 823-861)

**Looks in:** `~/.claude/commands/*.md` (global) or `<repoDir>/.claude/commands/*.md` (project).

**Parses:** YAML frontmatter for `name` and `description`.

### 10. Agents -- `scanAgents()` (lines 863-901)

**Looks in:** `~/.claude/agents/*.md` (global) or `<repoDir>/.claude/agents/*.md` (project).

**Parses:** YAML frontmatter for `name` and `description`.

### 11. Sessions -- `scanSessions()` (lines 903-967)

**Looks in:** `~/.claude/projects/<encoded>/*.jsonl` -- project scopes only. Returns an empty array for the global scope.

**Parses:** JSONL files using **chunked I/O** for performance. Session files can be very large, so the scanner avoids reading them entirely:
- `readFirstLines(path, 10)` (lines 54-78): reads the first 10 lines via 8KB buffer chunks to find the `aiTitle` field near the top of the file.
- `readLastLines(path, 30, fileSize)` (lines 80-110): reads the last 30 lines via reverse 8KB chunks to find the last user message, skipping XML-like content and `tool_use` entries.

Sessions are `deletable: true` but not movable -- they are inherently tied to the project scope where the conversation occurred.

## Why This Design Matters

Running all 11 scanners in parallel per scope keeps the full scan fast despite touching dozens of filesystem locations. The chunked I/O strategy for sessions is particularly important -- session JSONL files can grow to megabytes, and reading only the head and tail extracts the title and last message without loading the entire file into memory. The multi-source MCP server scanning ensures nothing is hidden, surfacing duplicates that would otherwise silently shadow each other across configuration files.
