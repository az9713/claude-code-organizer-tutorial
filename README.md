# Claude Code Organizer — Tutorial Edition

This repository is a clone of [claude-code-organizer](https://github.com/mcpware/claude-code-organizer) by mcpware, enhanced with a comprehensive tutorial that explains how the tool works internally and documents the AI agent team that produced it.

## Tutorial

All tutorial content lives in the [`tutorial/`](./tutorial/) folder:

| File | Description |
|------|-------------|
| [`how-it-works.md`](./tutorial/how-it-works.md) | **The main technical document** — a 9-section deep-dive into the codebase covering scope discovery, the 11-category scanner, the mover system, context budget calculator, dashboard UI, MCP server mode, plugin/skill system, and the full technology stack |
| [`cco-docs-team-documentation.md`](./tutorial/cco-docs-team-documentation.md) | **Agent team retrospective** — a complete turn-by-turn account of the 3-agent Claude Code team (researcher + writer + editor) that produced `how-it-works.md`, including every message, every task update, timestamps, and design decisions |
| [`docs_research_notes.md`](./tutorial/docs_research_notes.md) | Raw research notes produced by the researcher agent after reading ~6,000 lines of source code |
| `docs_draft_section_N.md` | First drafts of each section (before editor critique) |
| `docs_final_section_N.md` | Approved final versions of each section (after critique and revision) |

## How the Tutorial Was Created

The tutorial was produced by a **Claude Code agent team** — a coordinated team of 3 AI agents orchestrated by Claude Code's team system:

- **researcher** — read all source files (~6,000 lines across 12 files) and produced structured reference notes
- **writer** — drafted all 9 documentation sections one at a time, revising after each editor critique
- **editor** — cross-checked every draft against the source code and research notes, catching 7 categories of inaccuracies before approval

The agents communicated via `SendMessage`, coordinated work through a shared task list with dependency chains, and handed off content through the filesystem. Zero orchestrator interventions were needed during the 18-cycle write-edit loop.

The complete account — turn by turn, message by message, timestamp by timestamp — is in [`cco-docs-team-documentation.md`](./tutorial/cco-docs-team-documentation.md).

## Security Audit & Remediation

This repository also contains a complete OWASP security audit and all 18 remediations, applied to the codebase by a 5-agent Claude Code team.

| File | Description |
|------|-------------|
| [`SECURITY-AUDIT.md`](./SECURITY-AUDIT.md) | **Full OWASP Top 10 audit** — beginner-friendly report covering 18 findings (1 Critical, 6 High, 6 Medium, 4 Low, 6 Informational), attack chain analysis, CVSS scores, vulnerable code snippets, attack scenarios, and suggested fixes |
| [`SECURITY-REMEDIATION-REPORT.md`](./SECURITY-REMEDIATION-REPORT.md) | **Remediation documentation** — per-vulnerability before/after code for all 18 fixes, agent team architecture, per-agent inputs/outputs, and a turn-by-turn communication log |
| [`docs/plans/2026-03-26-owasp-security-remediations.md`](./docs/plans/2026-03-26-owasp-security-remediations.md) | **TDD implementation plan** — the step-by-step plan (red-green-refactor) used to drive the fixes |

### What Was Fixed

| Severity | Count | Examples |
|----------|-------|---------|
| Critical | 1 | Path traversal via `/../` in file restore endpoint |
| High | 6 | CSRF, MCP API key exposure, shell injection, missing auth headers |
| Medium | 6 | Prototype pollution, TOCTOU race, body size limit, open network binding |
| Low | 4 | Error message leakage, missing SRI hash, Google Fonts data exfiltration |
| Informational | 6 | Dependency audit, response header hygiene, CSP hardening |

### How the Fixes Were Applied

A **5-agent Claude Code team** implemented all changes in parallel — each agent owned a non-overlapping set of files to avoid merge conflicts:

- **server-agent** — `src/server.mjs`: CSRF, path traversal, security headers, CSP, body limit, network binding
- **scanner-agent** — `src/scanner.mjs`: MCP API key redaction, prototype pollution
- **mover-agent** — `src/mover.mjs`: TOCTOU race conditions, prototype pollution
- **frontend-agent** — `src/ui/index.html`, `src/ui/style.css`, `src/ui/app.js`: SRI hash, font privacy, HTTP error handling
- **cli-agent** — `bin/cli.mjs`: shell injection prevention via `execFile`

The full turn-by-turn communication between agents is documented in [`SECURITY-REMEDIATION-REPORT.md`](./SECURITY-REMEDIATION-REPORT.md).

## About the Original Project

**Claude Code Organizer** is a visual configuration manager for Claude Code's `~/.claude/` directory. It scans your memories, skills, MCP servers, hooks, plans, rules, commands, agents, sessions, and plugins across all scopes — and lets you move or delete them through a web dashboard or MCP server interface.

- Original repository: [mcpware/claude-code-organizer](https://github.com/mcpware/claude-code-organizer)
- Install: `npx @mcpware/claude-code-organizer`

## License

Original project license applies — see [LICENSE](./LICENSE).
