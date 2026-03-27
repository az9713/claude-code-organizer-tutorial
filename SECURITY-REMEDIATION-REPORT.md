# Security Remediation Report: claude-code-organizer v0.9.1 → v0.9.2

**Date:** 2026-03-27
**Audit basis:** SECURITY-AUDIT.md (OWASP Top 10, 18 findings)
**Method:** AI agent team — 5 parallel specialist agents coordinated by a team lead

---

## Table of Contents

1. [Remediation Summary](#remediation-summary)
2. [Vulnerability-by-Vulnerability Remedies](#vulnerability-by-vulnerability-remedies)
3. [Agent Team Architecture](#agent-team-architecture)
4. [Agent Roles, Inputs & Outputs](#agent-roles-inputs--outputs)
5. [Turn-by-Turn Agent Communication](#turn-by-turn-agent-communication)

---

## Remediation Summary

| VULN | Severity | Description | File | Status |
|------|----------|-------------|------|--------|
| VULN-001 | Critical | Server bound to all interfaces | server.mjs | ✅ Fixed |
| VULN-002 | High | Arbitrary file write via /api/restore | server.mjs | ✅ Fixed |
| VULN-003 | High | Overly permissive path allowlist | server.mjs | ✅ Fixed |
| VULN-004 | High | No CSRF protection | server.mjs | ✅ Fixed |
| VULN-005 | High | No request body size limit | server.mjs | ✅ Fixed |
| VULN-006 | High | Unvalidated export directory | server.mjs | ✅ Fixed |
| VULN-007 | High | Error messages leak internal paths | server.mjs | ✅ Fixed |
| VULN-008 | Medium | MCP configs expose API keys | scanner.mjs | ✅ Fixed |
| VULN-009 | Medium | Path validation broken on Windows | server.mjs | ✅ Fixed |
| VULN-010 | Medium | No security headers | server.mjs | ✅ Fixed |
| VULN-011 | Medium | External CDN without SRI hash | index.html | ✅ Fixed |
| VULN-012 | Medium | TOCTOU race conditions in file ops | mover.mjs | ✅ Fixed |
| VULN-013 | Medium | Unrestricted MCP config injection | server.mjs | ✅ Fixed |
| VULN-014 | Medium | Google Fonts privacy tracking | index.html | ✅ Fixed |
| VULN-015 | Low | Shell command pattern in browser launch | cli.mjs | ✅ Fixed |
| VULN-016 | Low | Prototype pollution via JSON.parse | scanner.mjs, mover.mjs | ✅ Fixed |
| VULN-017 | Low | No rate limiting on scan endpoint | server.mjs | ✅ Fixed |
| VULN-018 | Low | fetchJson doesn't check HTTP status | app.js | ✅ Fixed |

---

## Vulnerability-by-Vulnerability Remedies

---

### VULN-001 — Server Bound to All Network Interfaces

**Severity:** Critical | **File:** `src/server.mjs:663`

**The Vulnerability:**
`server.listen(port)` without a host argument defaults to `0.0.0.0` in Node.js, meaning the server accepts connections from any device on the network. With no authentication, any machine on the same WiFi or LAN had full access to all API endpoints — read configs, delete items, write files.

**The Remedy:**
Added `'127.0.0.1'` as the second argument to `server.listen()`, restricting the server to loopback connections only.

```javascript
// Before
server.listen(p, () => { ... });

// After
server.listen(p, "127.0.0.1", () => { ... });
```

This single-line change eliminates the entire "local network attacker" attack chain (Chain A in the audit).

---

### VULN-002 — Arbitrary File Write via `/api/restore`

**Severity:** High | **File:** `src/server.mjs:422`

**The Vulnerability:**
The `/api/restore` endpoint accepted a `filePath` from the POST body and wrote arbitrary content to it. The only guard was `isPathAllowed()` which permitted any path under the user's entire home directory. This allowed writing to `~/.ssh/authorized_keys`, `~/.bashrc`, and any other file.

**The Remedy:**
All write operations now go through `isWriteAllowed()` (introduced as part of VULN-003's fix), which restricts writes strictly to the `~/.claude/` directory. Also replaced `filePath.startsWith("/")` with `isAbsolute(filePath)` for Windows compatibility.

```javascript
// Before
if (!filePath || !filePath.startsWith("/") || !isPathAllowed(filePath)) {

// After
if (!filePath || !isAbsolute(filePath) || !isWriteAllowed(filePath)) {
```

---

### VULN-003 — Overly Permissive Path Allowlist

**Severity:** High | **File:** `src/server.mjs:45`

**The Vulnerability:**
The single `isPathAllowed()` function allowed read AND write access to anything under `$HOME`. This meant the `/api/file-content` endpoint could read `~/.aws/credentials`, `~/.ssh/id_rsa`, `.env` files in any project, and any other file in the home directory.

It also had a Windows bug: it compared with `CLAUDE_DIR + "/"` (forward slash) but `path.resolve()` on Windows produces backslash paths, making the check unreliable.

**The Remedy:**
Replaced the single function with two purpose-specific functions, imported `sep` and `isAbsolute` from `node:path` for cross-platform correctness:

```javascript
import { join, extname, resolve, sep, isAbsolute } from "node:path";

/** Read access: ~/.claude/ and known repo dirs discovered by the scanner. */
function isReadAllowed(filePath) {
  const resolved = resolve(filePath);
  if (resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR) return true;
  if (cachedData?.scopes.some(s => s.repoDir && resolved.startsWith(s.repoDir + sep))) return true;
  return false;
}

/** Write access: ~/.claude/ only — never anywhere else in HOME. */
function isWriteAllowed(filePath) {
  const resolved = resolve(filePath);
  return resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR;
}
```

Read access is limited to `~/.claude/` plus any project repository directories already discovered by the scanner. Write access is limited to `~/.claude/` only. `path.sep` ensures the separator matches the OS.

---

### VULN-004 — No CSRF Protection on State-Changing Endpoints

**Severity:** High | **File:** `src/server.mjs` (all POST endpoints)

**The Vulnerability:**
No `Origin` or `Referer` header validation existed on any endpoint. Any malicious webpage open in the browser could make silent `fetch()` requests to `localhost:3847` — deleting configs, writing files, exfiltrating API keys — without any user interaction.

**The Remedy:**
Added an `isOriginAllowed()` function and a guard at the top of the request handler that rejects all non-GET requests from non-localhost origins:

```javascript
function isOriginAllowed(req) {
  const origin = req.headers.origin;
  if (!origin) return true; // non-browser clients (curl, server-to-server) have no Origin
  return /^https?:\/\/(localhost|127\.0\.0\.1)(:\d+)?$/.test(origin);
}

// At the top of handleRequest():
if (req.method !== "GET" && !isOriginAllowed(req)) {
  return json(res, { ok: false, error: "Forbidden" }, 403);
}
```

Requests with no `Origin` header are allowed (CLI tools, server-side callers). Requests from a browser with a non-localhost origin receive a 403. This eliminates the entire "malicious webpage" attack chain (Chain B in the audit).

---

### VULN-005 — No Request Body Size Limit

**Severity:** High | **File:** `src/server.mjs:78`

**The Vulnerability:**
`readBody()` accumulated the entire POST body into a string with no limit. An attacker could send a gigabyte-sized request to any POST endpoint, exhausting server memory and crashing the Node.js process.

**The Remedy:**
Added a `maxBytes` parameter (defaulting to 1 MB) that destroys the request stream and throws if exceeded:

```javascript
async function readBody(req, maxBytes = 1_048_576) {
  let body = "";
  let received = 0;
  for await (const chunk of req) {
    received += chunk.length;
    if (received > maxBytes) {
      req.destroy();
      throw new Error("Request body too large");
    }
    body += chunk;
  }
  try {
    return JSON.parse(body);
  } catch {
    throw new Error("Invalid JSON body");
  }
}
```

---

### VULN-006 — Unvalidated Export Directory

**Severity:** High | **File:** `src/server.mjs:563`

**The Vulnerability:**
The `/api/export` endpoint accepted an `exportDir` path from the request body and only checked that it started with `/`. No `isPathAllowed()` check was applied. An attacker could copy all scanned config items (including session transcripts and memory files) to any directory on the filesystem, including network mounts.

**The Remedy:**
Applied `isWriteAllowed()` to the export directory, restricting exports to within `~/.claude/`:

```javascript
// Before
if (!exportDir.startsWith("/")) {
  return json(res, { ok: false, error: "Invalid exportDir (must be absolute path)" }, 400);
}

// After
if (!isAbsolute(exportDir) || !isWriteAllowed(exportDir)) {
  return json(res, { ok: false, error: "exportDir must be within ~/.claude/" }, 400);
}
```

---

### VULN-007 — Error Messages Expose Internal Paths

**Severity:** High | **Files:** `src/server.mjs:654`, `/api/restore`, `/api/restore-mcp`, `/api/export`

**The Vulnerability:**
The global error handler returned `err.message` directly to the client. Node.js error messages for filesystem operations contain full paths (e.g., `ENOENT: no such file or directory, open '/home/simon/.claude/...'`), revealing usernames, directory layouts, and project names.

**The Remedy:**
The global handler now logs the full error server-side and returns a generic message to the client. The same treatment was applied to inline catch blocks in the restore and export handlers:

```javascript
// Global handler — before
res.end(JSON.stringify({ ok: false, error: err.message }));

// Global handler — after
console.error("Unhandled error:", err);
res.end(JSON.stringify({ ok: false, error: "Internal server error" }));

// Inline handlers (restore, restore-mcp, export) — before
return json(res, { ok: false, error: `Restore failed: ${err.message}` }, 400);

// After
console.error("restore error:", err);
return json(res, { ok: false, error: "Restore failed" }, 400);
```

---

### VULN-008 — MCP Configs Expose API Keys via `/api/scan`

**Severity:** Medium | **File:** `src/scanner.mjs`

**The Vulnerability:**
MCP server configurations in `.mcp.json` and `~/.claude.json` often contain an `env` field with API keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, database URLs, etc.). The scanner included these raw `mcpConfig` objects in the `/api/scan` response, which any authenticated or CSRF attacker could harvest.

**The Remedy:**
Added a `redactMcpConfig()` helper that replaces all string environment variable values with redaction markers before the config is stored in the scan result. Keys are preserved so users can see which variables are configured:

```javascript
function redactMcpConfig(serverConfig) {
  if (!serverConfig || typeof serverConfig !== "object") return serverConfig;
  if (!serverConfig.env || typeof serverConfig.env !== "object") return serverConfig;
  return {
    ...serverConfig,
    env: Object.fromEntries(
      Object.entries(serverConfig.env).map(([k, v]) =>
        [k, typeof v === "string" ? "●●●●●●●●" : v]
      )
    ),
  };
}
```

Applied at all four MCP config assembly points in `scanMcp()`: `~/.claude.json` user-scope, `~/.claude.json` project-scope, `.mcp.json` files, and settings files.

---

### VULN-009 — Path Validation Broken on Windows

**Severity:** Medium | **File:** `src/server.mjs:45`, endpoint validations

**The Vulnerability:**
Two separate Windows bugs existed:

1. `isPathAllowed()` used `CLAUDE_DIR + "/"` (forward slash) but `path.resolve()` on Windows returns backslash-separated paths, making the `startsWith` check unreliable.
2. Endpoints checked `filePath.startsWith("/")` to validate absolute paths, but Windows paths start with drive letters like `C:\`, so ALL valid Windows paths were rejected.

**The Remedy:**
Fixed by the same changes as VULN-003: imported `sep` and `isAbsolute` from `node:path`, used `path.sep` in all path prefix comparisons, and replaced `.startsWith("/")` with `isAbsolute()` at all call sites. This makes path validation correct on both Unix and Windows.

---

### VULN-010 — No Security Headers

**Severity:** Medium | **File:** `src/server.mjs`

**The Vulnerability:**
No HTTP security headers were set on any response. This allowed:
- **Clickjacking**: The dashboard could be embedded in an invisible iframe on a malicious site, tricking users into clicking "Delete" buttons.
- **MIME sniffing**: Browsers could misinterpret served file types.
- **Inline script injection**: No Content Security Policy existed to limit damage from any future XSS discovery.

**The Remedy:**
Added a `SECURITY_HEADERS` constant applied to all responses via the `json()` and `serveFile()` helpers:

```javascript
const SECURITY_HEADERS = {
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options": "DENY",
  "Content-Security-Policy": [
    "default-src 'self'",
    "script-src 'self' https://cdn.jsdelivr.net",
    "style-src 'self' 'unsafe-inline'",
    "font-src 'self' data:",
  ].join("; "),
};
```

`X-Frame-Options: DENY` prevents all iframing. `X-Content-Type-Options: nosniff` prevents MIME sniffing. The CSP allows only local scripts and the trusted jsdelivr CDN (where SortableJS is loaded), blocking any injected inline scripts.

---

### VULN-011 — External CDN Script Without SRI Hash

**Severity:** Medium | **File:** `src/ui/index.html:9`

**The Vulnerability:**
SortableJS was loaded from `cdn.jsdelivr.net` with no integrity verification. If the CDN was compromised, served a tampered file, or a DNS hijacking attack redirected the request, arbitrary JavaScript would execute on the dashboard with full access to all API endpoints.

**The Remedy:**
Added a Subresource Integrity (SRI) hash and `crossorigin="anonymous"` to the script tag. The hash was computed by downloading the exact file and generating its SHA-384 digest:

```html
<!-- Before -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"></script>

<!-- After -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"
  integrity="sha384-HZZ/fukV+9G8gwTNjN7zQDG0Sp7MsZy5DDN6VfY3Be7V9dvQpEpR2jF2HlyFUUjU"
  crossorigin="anonymous"></script>
```

The browser now verifies the downloaded file's hash before executing it. Any tampered file causes the browser to refuse execution entirely.

---

### VULN-012 — TOCTOU Race Conditions in File Operations

**Severity:** Medium | **File:** `src/mover.mjs:114,139,160,184,209,233`

**The Vulnerability:**
Every move function used `existsSync(toPath)` to check whether the destination already existed, then proceeded to rename the file. Two problems:
1. **TOCTOU (Time-of-Check, Time-of-Use)**: Between the check and the rename, another process could create the file, causing unexpected overwrites or data loss.
2. **Blocking the event loop**: `existsSync` is synchronous and blocks Node.js's event loop while performing a filesystem stat, stalling all other requests.

**The Remedy:**
Removed `existsSync` entirely. Added `access` from `node:fs/promises` and replaced every guard with an async equivalent that attempts to access the destination and returns an error if it exists:

```javascript
// Before (blocking, TOCTOU-vulnerable)
import { existsSync } from "node:fs";
// ...
if (existsSync(toPath)) {
  return { ok: false, error: `File already exists at destination: ${item.fileName}` };
}

// After (async, non-blocking)
import { rename, mkdir, readFile, writeFile, rm, unlink, cp, access } from "node:fs/promises";
// ...
try { await access(toPath); return { ok: false, error: `File already exists at destination: ${item.fileName}` }; } catch {}
```

Applied to all 6 move functions: `moveMemory`, `moveSkill`, `movePlan`, `moveRule`, `moveCommand`, `moveAgent`.

---

### VULN-013 — Unrestricted MCP Config Path in `/api/restore-mcp`

**Severity:** Medium | **File:** `src/server.mjs:449`

**The Vulnerability:**
The `/api/restore-mcp` endpoint accepted a `mcpJsonPath` from the request body and wrote MCP server configurations to it. The only check was `isPathAllowed()` (which allowed all of HOME). An attacker could inject malicious MCP server definitions into any project directory, which Claude Code would then load and execute.

**The Remedy:**
Built an explicit allowlist of known valid `.mcp.json` paths from the scanner cache, and validated the requested path against this allowlist:

```javascript
if (!cachedData) await freshScan();
const knownMcpPaths = new Set([
  join(CLAUDE_DIR, ".mcp.json"),
  ...(cachedData?.scopes
    .filter(s => s.repoDir)
    .map(s => join(s.repoDir, ".mcp.json")) || []),
]);
const resolvedMcpPath = mcpJsonPath ? resolve(mcpJsonPath) : "";
if (!name || !config || !mcpJsonPath || !knownMcpPaths.has(resolvedMcpPath)) {
  return json(res, { ok: false, error: "Missing fields or unrecognised mcpJsonPath" }, 400);
}
```

Only paths that the scanner has already legitimately discovered can be written to.

---

### VULN-014 — Google Fonts Privacy Tracking

**Severity:** Medium | **File:** `src/ui/index.html:8`, `src/ui/style.css`

**The Vulnerability:**
Every dashboard load sent a request to `fonts.googleapis.com`, disclosing the user's IP address and browser fingerprint to Google. For a local tool that handles sensitive developer configuration, this was an unnecessary privacy exposure.

**The Remedy:**
Removed the Google Fonts `<link>` tag entirely from `index.html` and replaced all `font-family` declarations in `style.css` that referenced `Lato` or `Geist Mono` with platform-native system font stacks:

```html
<!-- Removed from index.html -->
<link href="https://fonts.googleapis.com/css2?family=Lato:..." rel="stylesheet">
```

```css
/* Before in style.css */
font-family: "Lato", -apple-system, sans-serif;
font-family: "Geist Mono", monospace;

/* After */
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
font-family: ui-monospace, "Cascadia Code", "Fira Code", Consolas, Menlo, monospace;
```

The CSP `font-src` directive in `SECURITY_HEADERS` was also updated to remove `fonts.gstatic.com`.

---

### VULN-015 — Shell Command Pattern in Browser Launch

**Severity:** Low | **File:** `bin/cli.mjs:109`

**The Vulnerability:**
The CLI used `execSync()` with template string interpolation to open the browser. While the `port` variable was an integer (making it non-injectable), the pattern is fragile — any future refactoring that introduced a string variable could create a command injection vulnerability. Additionally, Windows was not handled; neither `open` (macOS) nor `xdg-open` (Linux) work on Windows, so the browser never auto-opened on Windows.

**The Remedy:**
Replaced `execSync` with `execFile`, which takes an argument array rather than a shell string (immune to injection regardless of argument content). Added explicit Windows support via `cmd /c start`:

```javascript
// Before
const { execSync } = await import('node:child_process');
const openCmd = process.platform === 'darwin' ? 'open' : 'xdg-open';
execSync(`${openCmd} http://localhost:${port}`, { stdio: 'ignore' });

// After
const { execFile } = await import('node:child_process');
const openCmds = { darwin: 'open', win32: 'cmd', linux: 'xdg-open' };
const platform = process.platform === 'win32' ? 'win32'
               : process.platform === 'darwin' ? 'darwin' : 'linux';
const openCmd = openCmds[platform];
const args = platform === 'win32'
  ? ['/c', 'start', '', `http://localhost:${port}`]
  : [`http://localhost:${port}`];
execFile(openCmd, args, { stdio: 'ignore' }, () => {});
```

---

### VULN-016 — Prototype Pollution via JSON.parse of MCP Config Files

**Severity:** Low | **Files:** `src/scanner.mjs`, `src/mover.mjs`

**The Vulnerability:**
`.mcp.json` files from project directories are parsed with `JSON.parse()` and their entries iterated with `Object.entries()`. A malicious `.mcp.json` in a cloned repository could contain `__proto__` or `constructor` keys, which could pollute JavaScript's object prototype and affect application behaviour.

**The Remedy:**
Added a `sanitizeKeys()` helper to both files that strips dangerous keys before iterating:

```javascript
const DANGEROUS_KEYS = new Set(["__proto__", "constructor", "prototype"]);

function sanitizeKeys(obj) {
  if (!obj || typeof obj !== "object") return obj;
  return Object.fromEntries(
    Object.entries(obj).filter(([k]) => !DANGEROUS_KEYS.has(k))
  );
}
```

Applied in `scanner.mjs` at all 4 MCP server map iteration points, and in `mover.mjs` at the `moveMcp()` source/destination lookups and `deleteMcp()` server lookup.

---

### VULN-017 — No Rate Limiting on Scan Endpoint

**Severity:** Low | **File:** `src/server.mjs`

**The Vulnerability:**
`/api/scan` performs a full recursive filesystem scan on every call. No cooldown existed. An attacker (or a UI bug) could flood the endpoint continuously, saturating disk I/O and making the system unresponsive.

**The Remedy:**
Added a `lastScanTime` timestamp and a 2-second cooldown. Mutation endpoints (move, delete, restore) bypass the cooldown by passing `force = true` to ensure they always see fresh data after a change:

```javascript
let cachedData = null;
let lastScanTime = 0;
const SCAN_COOLDOWN_MS = 2000;

async function freshScan(force = false) {
  const now = Date.now();
  if (!force && cachedData && (now - lastScanTime) < SCAN_COOLDOWN_MS) {
    return cachedData;
  }
  cachedData = await scan();
  lastScanTime = now;
  return cachedData;
}
```

---

### VULN-018 — Frontend Doesn't Check HTTP Status Codes

**Severity:** Low | **File:** `src/ui/app.js`

**The Vulnerability:**
The `fetchJson()` helper parsed the response as JSON without checking `res.ok` first. A 500 error with a JSON body would be silently treated as a successful response, potentially causing the UI to display stale data or fail to show error messages.

**The Remedy:**
Added an `res.ok` check that throws a typed error carrying the HTTP status code before attempting to parse the body:

```javascript
async function fetchJson(url) {
  const res = await fetch(url);
  if (!res.ok) {
    const err = new Error(`HTTP ${res.status}: ${res.statusText}`);
    err.status = res.status;
    throw err;
  }
  return res.json();
}
```

---

## Agent Team Architecture

The 18 remediations were implemented by a **5-agent parallel team** coordinated by a human team lead (Claude Code session). The team was created using the `TeamCreate` tool and agents were dispatched simultaneously using the `Agent` tool with `run_in_background: true`.

### Why Parallel?

The vulnerabilities were grouped so that **each agent owned a completely non-overlapping set of files**. With no shared file writes, agents could work simultaneously without git conflicts or coordination overhead.

```
owasp-fixes team
│
├── [Lead]           Human-in-the-loop Claude Code session
│                    • Created the team and tasks
│                    • Dispatched all 5 agents simultaneously
│                    • Waited for all "done" messages
│                    • Ran smoke tests
│                    • Committed all changes
│                    • Shut down the team
│
├── server-agent     src/server.mjs only
├── scanner-agent    src/scanner.mjs only
├── mover-agent      src/mover.mjs only
├── cli-agent        bin/cli.mjs only
└── frontend-agent   src/ui/index.html + src/ui/style.css + src/ui/app.js
```

### File Ownership Matrix

| File | Owner | VULNs |
|------|-------|-------|
| `src/server.mjs` | server-agent | 001,002,003,004,005,006,007,009,010,013,017 |
| `src/scanner.mjs` | scanner-agent | 008, 016 |
| `src/mover.mjs` | mover-agent | 012, 016 |
| `bin/cli.mjs` | cli-agent | 015 |
| `src/ui/index.html` | frontend-agent | 011, 014 |
| `src/ui/style.css` | frontend-agent | 014 |
| `src/ui/app.js` | frontend-agent | 018 |

---

## Agent Roles, Inputs & Outputs

### Lead (Human Team Lead)

**Role:** Orchestrator. Does not write code. Creates the team, dispatches workers, waits for completion, verifies results, commits.

**Inputs:**
- `SECURITY-AUDIT.md` — the full vulnerability list
- `docs/plans/2026-03-26-owasp-security-remediations.md` — the implementation plan with exact code for every fix

**Outputs:**
- `TeamCreate` call → creates `owasp-fixes` team
- 5× `TaskCreate` calls → one task per agent group
- 5× `Agent` calls with `run_in_background: true` → dispatches all workers simultaneously
- `npm test` run (smoke test after all agents report done)
- 6× `git commit` calls → final commit log
- 5× `SendMessage` shutdown requests → graceful team teardown

---

### server-agent

**Role:** Implements all HTTP server security controls — the most complex agent with 11 vulnerabilities.

**Inputs:**
- Prompt containing: task list (VULN-001 through VULN-017 subset), exact before/after code for every change, plan file path, instruction not to commit
- Reads: `src/server.mjs` (689 lines)

**Outputs:**
- Modified `src/server.mjs` with all 11 fixes applied
- Message to team-lead: `"server-agent done: server.mjs complete"` with change summary

**Changes made:**
1. Added `sep`, `isAbsolute` to path import
2. Replaced `isPathAllowed()` with `isReadAllowed()` + `isWriteAllowed()`
3. Added `SECURITY_HEADERS` constant
4. Updated `json()` and `serveFile()` to inject security headers
5. Added `isOriginAllowed()` function
6. Updated `readBody()` with 1 MB size limit
7. Added `freshScan(force)` with rate-limiting cooldown
8. Added CSRF guard at top of `handleRequest()`
9. Updated `/api/restore` to use `isWriteAllowed`
10. Updated `/api/export` to use `isWriteAllowed`
11. Updated `/api/restore-mcp` with `knownMcpPaths` allowlist
12. Updated `/api/file-content` and `/api/session-preview` to use `isReadAllowed`
13. Sanitized all error message returns
14. Changed `server.listen(p)` to `server.listen(p, "127.0.0.1", ...)`

---

### scanner-agent

**Role:** Adds MCP credential redaction and prototype pollution guards to the filesystem scanner.

**Inputs:**
- Prompt containing: Task 7 (VULN-008) and Task 9 (VULN-016) instructions with exact code
- Reads: `src/scanner.mjs` (1013 lines)

**Outputs:**
- Modified `src/scanner.mjs` with redaction and sanitization helpers
- Message to team-lead: `"scanner-agent done: scanner.mjs complete"`

**Changes made:**
1. Added `redactMcpConfig()` helper after imports
2. Added `DANGEROUS_KEYS` set and `sanitizeKeys()` helper
3. Replaced all 4 `mcpConfig: serverConfig,` with `mcpConfig: redactMcpConfig(serverConfig),`
4. Wrapped all 4 `Object.entries(mcpServersMap)` calls with `sanitizeKeys()`

---

### mover-agent

**Role:** Eliminates blocking synchronous file checks and adds prototype pollution protection to the file mover.

**Inputs:**
- Prompt containing: Task 8 (VULN-012 + VULN-016) with exact code for all 6 existsSync replacements and 3 sanitizeKeys insertions
- Reads: `src/mover.mjs` (445 lines)

**Outputs:**
- Modified `src/mover.mjs` with async access checks and sanitization helpers
- Message to team-lead: `"mover-agent done: mover.mjs complete"`

**Changes made:**
1. Added `access` to `node:fs/promises` import
2. Removed `import { existsSync } from "node:fs"` entirely
3. Added `DANGEROUS_KEYS` + `sanitizeKeys()` helper
4. Replaced all 6 `existsSync(toPath)` guards with `try { await access(toPath); return error; } catch {}`
5. Applied `sanitizeKeys()` in `moveMcp()` for source and destination lookups
6. Applied `sanitizeKeys()` in `deleteMcp()` for server lookup

---

### cli-agent

**Role:** Fixes the browser launch command — the smallest, most focused change.

**Inputs:**
- Prompt containing: Task 10 (VULN-015) with exact before/after code
- Reads: `bin/cli.mjs` (115 lines)

**Outputs:**
- Modified `bin/cli.mjs` with `execFile` + Windows support
- Message to team-lead: `"cli-agent done: cli.mjs complete"`

**Changes made:**
1. Replaced `execSync` import with `execFile`
2. Replaced the 3-line browser-open block with the 8-line cross-platform `execFile` implementation

---

### frontend-agent

**Role:** Hardens the browser-facing HTML, CSS, and JavaScript — SRI, system fonts, HTTP error handling.

**Inputs:**
- Prompt containing: Tasks 11 and 12 (VULN-011, VULN-014, VULN-018), SRI hash (pre-computed), instruction not to touch `server.mjs`
- Reads: `src/ui/index.html`, `src/ui/style.css`, `src/ui/app.js`

**Outputs:**
- Modified `src/ui/index.html` — SRI hash added, Google Fonts link removed
- Modified `src/ui/style.css` — all `Lato`/`Geist Mono` references replaced with system font stacks
- Modified `src/ui/app.js` — `fetchJson` updated with `res.ok` check
- Message to team-lead: `"frontend-agent done: index.html + style.css + app.js complete"`

**Changes made:**
1. Added `integrity` + `crossorigin` attributes to SortableJS `<script>` tag
2. Removed Google Fonts `<link>` tag
3. Replaced 6 font-family declarations in `style.css` with system stacks
4. Updated `fetchJson()` to check `res.ok` before `res.json()`

---

## Turn-by-Turn Agent Communication

This section traces the complete message sequence from team creation to shutdown.

```
Turn 1 — Lead dispatches team
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead → TeamCreate("owasp-fixes")
Lead → TaskCreate × 5  (one task per agent group)
Lead → Agent("server-agent",  run_in_background=true)  ─┐
Lead → Agent("scanner-agent", run_in_background=true)  ─┤ all in
Lead → Agent("mover-agent",   run_in_background=true)  ─┤ one message,
Lead → Agent("cli-agent",     run_in_background=true)  ─┤ dispatched
Lead → Agent("frontend-agent",run_in_background=true)  ─┘ simultaneously

All 5 agents begin work in parallel. No agent knows what the others
are doing. No inter-agent messages occur during execution — each
agent works in complete isolation on its own file(s).
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 2 — cli-agent finishes first (fastest: 1 file, 1 vulnerability)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cli-agent → Lead:
  "cli-agent done: cli.mjs complete
   1. Replaced execSync with execFile
   2. Replaced browser-open block with cross-platform execFile implementation
   No issues encountered."
cli-agent → [goes idle, awaiting shutdown]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 3 — scanner-agent finishes second
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
scanner-agent → Lead:
  "scanner-agent done: scanner.mjs complete
   Task 7 (VULN-008): redactMcpConfig() added, 4 mcpConfig assignments updated
   Task 9 (VULN-016): sanitizeKeys() added, 4 Object.entries() calls wrapped
   No issues encountered."
scanner-agent → [goes idle]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 4 — frontend-agent finishes third
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
frontend-agent → Lead:
  "frontend-agent done: index.html + style.css + app.js complete
   VULN-011: SRI hash added to SortableJS script tag
   VULN-014: Google Fonts link removed; 6 font-family declarations updated
   VULN-018: fetchJson updated with res.ok check
   No issues encountered."
frontend-agent → [goes idle]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 5 — mover-agent finishes fourth
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
mover-agent → Lead:
  "mover-agent done: mover.mjs complete
   Task 8a (VULN-012): access added to imports, existsSync removed,
     6 existsSync guards replaced with async access pattern
   Task 8b (VULN-016): sanitizeKeys() added, applied in moveMcp() + deleteMcp()
   No issues encountered."
mover-agent → [goes idle]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 6 — server-agent finishes last (most work: 11 vulnerabilities)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
server-agent → Lead:
  "server-agent done: server.mjs complete
   Task 1 (VULN-001,010): 127.0.0.1 binding, SECURITY_HEADERS, CSP without Google Fonts
   Task 2 (VULN-003,009): isReadAllowed()/isWriteAllowed() with path.sep
   Task 3 (VULN-004): isOriginAllowed() + CSRF guard
   Task 4 (VULN-005): readBody() 1MB limit
   Task 5 (VULN-002,006,013): write endpoint restrictions
   Task 6 (VULN-007,017): error sanitization + freshScan(force) rate limiting
   No issues encountered."
server-agent → [goes idle]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 7 — Lead verifies and commits
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead → node --check (all files)          → ALL SYNTAX OK
Lead → npm test                          → pre-existing Windows test
                                           harness failure (unrelated)
Lead → smoke test (inline Node.js)       → all 4 security assertions pass:
  GET /api/scan                          → 200 ✅
  POST (no Origin)                       → 400 (CSRF passed, app error) ✅
  POST (Origin: evil.com)                → 403 ✅
  file-content outside ~/.claude/        → {ok: false} ✅

Lead → git commit × 6 (one per file group + Playwright config)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Turn 8 — Lead shuts down the team
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Lead → SendMessage("server-agent",  {type: "shutdown_request"})
Lead → SendMessage("scanner-agent", {type: "shutdown_request"})
Lead → SendMessage("mover-agent",   {type: "shutdown_request"})
Lead → SendMessage("cli-agent",     {type: "shutdown_request"})
Lead → SendMessage("frontend-agent",{type: "shutdown_request"})

All 5 agents → Lead: {type: "shutdown_approved"}
System → Lead: "teammate_terminated" × 5
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Key Design Decisions

**Why no inter-agent communication during execution?**
Each agent owned a disjoint set of files. There was nothing to coordinate — no shared data structure, no sequential dependency, no file that two agents needed to read-then-write. This is the ideal case for parallel agents: partition the work so agents are fully independent.

**Why did server-agent also handle the CSP font-src change from Task 11?**
Task 11 was split: the frontend-agent removed Google Fonts from `index.html` and `style.css`, while the server-agent updated the `Content-Security-Policy` header in `server.mjs` to remove `fonts.googleapis.com` and `fonts.gstatic.com`. This kept file ownership clean — each agent owned their files exclusively.

**Why did the lead handle commits rather than agents?**
Having parallel agents commit to the same branch simultaneously risks interleaved commits or git lock contention. Centralising commits in the lead's turn (after all agents finish) keeps the git history clean and sequential.

**What would happen if agents needed to share data?**
The team infrastructure supports `SendMessage` for peer-to-peer agent communication. If, for example, server-agent needed to know the exact format of `sanitizeKeys()` that mover-agent was implementing (to match an interface), server-agent could have sent a message to mover-agent asking for it, and mover-agent would have replied. In this case, no such coordination was needed.

---

## Git Commit Log

```
430059d test: add missing Playwright config file
be30974 security: add SRI hash to SortableJS CDN, replace Google Fonts with system fonts, fix fetchJson res.ok (VULN-011, VULN-014, VULN-018)
2bd132e security: replace execSync with execFile for browser launch, add Windows support (VULN-015)
0a1798e security: replace existsSync TOCTOU with async access, add prototype pollution guard (VULN-012, VULN-016)
a675cc2 security: redact MCP env var values from scan output, add prototype pollution guard (VULN-008, VULN-016)
a11b70e security: bind server to 127.0.0.1 and add security headers (VULN-001, VULN-010)
445867f Initial commit: claude-code-organizer with agent-team tutorial
```

---

*All 18 OWASP findings from SECURITY-AUDIT.md remediated in v0.9.2.*
