# Plugin & Skill System

CCO integrates with Claude Code through two complementary mechanisms: a plugin package for the Claude Code plugin registry, and a standalone skill that auto-installs on first dashboard launch. Both approaches give users a way to invoke CCO from within Claude Code, but they serve different distribution paths -- the plugin bundles everything for registry-based installation, while the auto-installed skill provides a lightweight shortcut for `npx` users.

## `.claude-plugin/` Package Structure

The `.claude-plugin/` directory at the project root makes CCO installable as a Claude Code plugin. It contains three files:

### `plugin.json` -- Plugin Metadata

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

### `settings.json` -- Auto-Configuration

When the plugin is installed, this file automatically configures two things:

1. **MCP server registration**: adds a `claude-code-organizer` entry to the user's MCP server config, configured to run via `npx @mcpware/claude-code-organizer --mcp`.
2. **Bash permission**: grants `Bash(npx @mcpware/claude-code-organizer*)` so Claude Code can launch the dashboard without prompting for permission.

This means installing the plugin immediately enables both MCP server mode and dashboard access without manual configuration.

### `skills/organize.md` -- Plugin-Bundled Skill

A skill definition bundled with the plugin. Uses frontmatter to declare its interface:

```yaml
---
name: organize
description: Open the Claude Code Organizer dashboard to view and manage memories, skills, MCP servers, and hooks across all scopes.
argument-hint: "[port]"
---
```

The body contains instructions for Claude Code to run the dashboard via `npx`. Note that the plugin-bundled skill omits the `allowed-tools` frontmatter field -- tool permissions are managed by the plugin's `settings.json` instead.

## Auto-Install of `/cco` Skill (cli.mjs lines 36-61)

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
5. Silently catches and ignores write permission errors -- if the user's filesystem prevents the write, the dashboard still starts normally.

This runs only in dashboard mode (not MCP mode) and only on the first launch. Subsequent launches detect the existing file and skip the install.

## Standalone Skill Definition

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

## Skill Definition Format

Claude Code skills follow a consistent format:

- **Frontmatter** (YAML between `---` delimiters): declares `name`, `description`, and optional fields like `argument-hint` and `allowed-tools`.
- **Body** (markdown after the frontmatter): contains the instructions that Claude Code follows when the skill is invoked. For CCO's skills, this instructs Claude Code to run `npx @mcpware/claude-code-organizer` with appropriate arguments.

Skills are discovered by scanning `SKILL.md` files in subdirectories of `~/.claude/skills/` (global) or `<repoDir>/.claude/skills/` (project-scoped). The scanner extracts the description from the first meaningful paragraph line after the heading, not from the frontmatter -- a detail covered in Section 3.

## Why This Design Matters

The dual distribution strategy -- plugin registry and auto-installed skill -- ensures CCO is accessible regardless of how a user discovers it. Plugin users get automatic MCP server setup and bash permissions. `npx` users get a `/cco` slash command on first launch. Both paths converge on the same functionality. The auto-install's graceful failure mode (silently catching permission errors) means the dashboard never fails to start just because skill installation was blocked -- it treats the skill as a convenience, not a dependency.
