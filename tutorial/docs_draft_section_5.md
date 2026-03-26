# Context Budget Calculator

The context budget calculator in `src/server.mjs` answers a practical question: how many tokens does Claude Code consume with your current configuration before you type a single word? Every CLAUDE.md file, every rule, every skill description, every MCP tool schema takes up space in the context window. The budget calculator tokenizes all of it, walks the scope inheritance chain, and returns a detailed breakdown so developers can see exactly where their context is going.

## What Gets Counted

The calculator splits items into two categories based on how Claude Code loads them.

### Always-Loaded Items (lines 156-176)

These are present in every conversation, unconditionally:

- **System prompt**: ~6,500 tokens (estimated constant)
- **System tools (loaded part)**: ~6,000 tokens (estimated constant)
- **CLAUDE.md files**: full content from `CLAUDE.md`, `.claude/CLAUDE.md`, and managed `CLAUDE.md` -- read and tokenized
- **Rules** (`.claude/rules/*.md`): all rule files, full content
- **MEMORY.md**: first 200 lines or 25KB, whichever comes first (lines 267-268)
- **Skill descriptions**: the name and frontmatter description of each skill (not the full SKILL.md body)
- **Commands and agents**: full file content of all `.md` files in commands and agents directories

The always-loaded categories are: `skill`, `rule`, `command`, and `agent` (line 176). Config items included are specifically `CLAUDE.md`, `.claude/CLAUDE.md`, and `CLAUDE.md (managed)` (line 178).

### Deferred Items (lines 165-173)

These are reserved in the context window but only loaded on demand (typically via ToolSearch):

- **System tools (deferred part)**: ~10,500 tokens (estimated constant)
- **MCP tool definitions**: ~3,100 tokens per unique MCP server (line 308)

For MCP servers, Claude Code deduplicates by name with priority order: local > project > user. The budget calculator counts unique server names, not total entries across all config files, because duplicates shadow each other and only one instance consumes context.

## Scope Chain Inheritance (lines 146-153)

The calculator does not just count items in the requested scope -- it walks the full parent chain by following `parentId` links up to the global scope. For example, requesting the budget for a project scope collects:

1. Items from the project scope itself (current scope)
2. Items from the workspace scope, if any (inherited)
3. Items from the global scope (inherited)

Current-scope and inherited items are tokenized separately, so the API response distinguishes between "your project's own budget" and "what you inherit from parent scopes."

## Token Counting (tokenizer.mjs)

The `countTokens()` function (tokenizer.mjs lines 45-49) uses lazy initialization (lines 19-38): on first call, it attempts to import `ai-tokenizer` and `ai-tokenizer/encoding` with Claude's encoding. If the optional dependency is not installed, it falls back to:

```javascript
Math.ceil(Buffer.byteLength(text, "utf-8") / 4)
```

This estimation is roughly 75-85% accurate compared to the real tokenizer. The function returns `{ tokens, confidence }` where confidence is either `"measured"` (real tokenizer) or `"estimated"` (byte fallback).

## System Overhead Constants (lines 293-296)

The calculator includes fixed overhead that comes from Claude Code's own system setup:

```javascript
const SYSTEM_LOADED = 12500;   // system prompt (~6.5K) + loaded system tools (~6K)
const SYSTEM_DEFERRED = 10500; // deferred system tools
// MCP: mcpUniqueCount * 3100 tokens per unique server
```

These constants are hardcoded estimates based on measured values of Claude Code's system prompt and tool definitions.

## API Endpoint

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

## Why This Design Matters

Context budget visibility solves a real developer pain point: mysterious "context too long" errors or degraded Claude Code performance caused by accumulated configuration. By tokenizing everything before the conversation starts and presenting the breakdown by scope and category, the calculator lets developers identify which CLAUDE.md file, which set of rules, or which MCP server fleet is consuming the most budget -- and then use the mover system to reorganize items to the scopes where they are actually needed.
