# OWASP Security Remediations Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement all 18 security fixes identified in SECURITY-AUDIT.md, covering the full compounding attack chain and all individual vulnerabilities.

**Architecture:** All fixes are additive or minimally invasive changes to existing files. No new modules are created. Changes are grouped by file to minimise context-switching. The E2E Playwright suite (`tests/e2e/dashboard.spec.mjs`) is the regression harness — run it after each task group. Security-specific assertions have no existing tests; manual verification steps are provided.

**Tech Stack:** Node.js ≥ 20, ESM `.mjs`, raw `node:http`, Playwright for E2E tests.

---

## Pre-flight

```bash
cd C:/Users/simon/Downloads/claude-code-organizer
npm test   # baseline — all tests must pass before starting
```

If tests fail before you touch anything, stop and report. Do not proceed.

---

## Task 1 — Bind Server to 127.0.0.1 + Add Security Headers (VULN-001, VULN-010)

**Files:** Modify `src/server.mjs`

These two changes are purely additive — no logic changes, so E2E tests should be unaffected.

**Step 1: Add security headers constant after the MIME map (around line 62)**

In `src/server.mjs`, after the `MIME` object, add:

```javascript
// ── Security headers ─────────────────────────────────────────────────
const SECURITY_HEADERS = {
  "X-Content-Type-Options": "nosniff",
  "X-Frame-Options": "DENY",
  "Content-Security-Policy": [
    "default-src 'self'",
    "script-src 'self' https://cdn.jsdelivr.net",
    "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
    "font-src 'self' https://fonts.gstatic.com data:",
  ].join("; "),
};
```

**Step 2: Inject headers into the `json()` helper (line 73)**

Replace:
```javascript
function json(res, data, status = 200) {
  res.writeHead(status, { "Content-Type": "application/json" });
  res.end(JSON.stringify(data));
}
```
With:
```javascript
function json(res, data, status = 200) {
  res.writeHead(status, { "Content-Type": "application/json", ...SECURITY_HEADERS });
  res.end(JSON.stringify(data));
}
```

**Step 3: Inject headers into `serveFile()` (line 88)**

Replace:
```javascript
async function serveFile(res, filePath) {
  try {
    const content = await readFile(filePath);
    const mime = MIME[extname(filePath)] || "application/octet-stream";
    res.writeHead(200, { "Content-Type": mime });
    res.end(content);
  } catch {
    res.writeHead(404);
    res.end("Not found");
  }
}
```
With:
```javascript
async function serveFile(res, filePath) {
  try {
    const content = await readFile(filePath);
    const mime = MIME[extname(filePath)] || "application/octet-stream";
    res.writeHead(200, { "Content-Type": mime, ...SECURITY_HEADERS });
    res.end(content);
  } catch {
    res.writeHead(404, SECURITY_HEADERS);
    res.end("Not found");
  }
}
```

**Step 4: Bind server to 127.0.0.1 (line 663)**

Replace:
```javascript
    server.listen(p, () => {
```
With:
```javascript
    server.listen(p, "127.0.0.1", () => {
```

**Step 5: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (Playwright connects to localhost so 127.0.0.1 binding is compatible).

**Step 6: Manual verification**

Start the server and confirm it does NOT bind to 0.0.0.0:
```bash
npm start &
sleep 2
netstat -an | grep 3847
# Expected: 127.0.0.1:3847  (NOT 0.0.0.0:3847)
kill %1
```

Open browser dev tools → Network tab → reload dashboard → inspect response headers on any request. Confirm `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` are present.

**Step 7: Commit**

```bash
git add src/server.mjs
git commit -m "security: bind server to 127.0.0.1 and add security headers (VULN-001, VULN-010)"
```

---

## Task 2 — Refactor Path Validation: Split Read/Write, Fix Windows Separator (VULN-003, VULN-009)

**Files:** Modify `src/server.mjs`

This task replaces the single `isPathAllowed()` with two purpose-specific functions and fixes the Windows path separator bug.

**Step 1: Update the `path` import at the top of server.mjs to include `sep` and `isAbsolute`**

Replace:
```javascript
import { join, extname, resolve } from "node:path";
```
With:
```javascript
import { join, extname, resolve, sep, isAbsolute } from "node:path";
```

**Step 2: Replace `isPathAllowed()` with two functions (lines 45–52)**

Replace the entire `isPathAllowed` function:
```javascript
function isPathAllowed(filePath) {
  const resolved = resolve(filePath);
  // Allow paths under ~/.claude/ or under any discovered project repoDir
  if (resolved.startsWith(CLAUDE_DIR + "/") || resolved === CLAUDE_DIR) return true;
  // Allow paths under HOME (covers repo dirs with .mcp.json, CLAUDE.md etc)
  if (resolved.startsWith(HOME + "/")) return true;
  return false;
}
```

With:
```javascript
/** Read access: ~/.claude/ and known repo dirs discovered by the scanner. */
function isReadAllowed(filePath) {
  const resolved = resolve(filePath);
  if (resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR) return true;
  // Allow reads from repo dirs already discovered by the scanner
  if (cachedData?.scopes.some(s => s.repoDir && resolved.startsWith(s.repoDir + sep))) return true;
  return false;
}

/** Write access: ~/.claude/ only — never anywhere else in HOME. */
function isWriteAllowed(filePath) {
  const resolved = resolve(filePath);
  return resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR;
}
```

**Step 3: Update all call sites to use the correct function**

There are four call sites. Replace each one:

**/api/restore (line ~424) — write operation:**
```javascript
// Before
if (!filePath || !filePath.startsWith("/") || !isPathAllowed(filePath)) {
// After
if (!filePath || !isAbsolute(filePath) || !isWriteAllowed(filePath)) {
```

**/api/restore-mcp (line ~449) — write operation:**
```javascript
// Before
if (!name || !config || !mcpJsonPath || !isPathAllowed(mcpJsonPath)) {
// After
if (!name || !config || !mcpJsonPath || !isWriteAllowed(mcpJsonPath)) {
```

**/api/file-content (line ~473) — read operation:**
```javascript
// Before
if (!filePath || !filePath.startsWith("/") || !isPathAllowed(filePath)) {
// After
if (!filePath || !isAbsolute(filePath) || !isReadAllowed(filePath)) {
```

**/api/session-preview (line ~487) — read operation:**
```javascript
// Before
if (!filePath || !filePath.endsWith(".jsonl") || !isPathAllowed(filePath)) {
// After
if (!filePath || !filePath.endsWith(".jsonl") || !isReadAllowed(filePath)) {
```

**Step 4: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (tests use real `~/.claude/` paths which are inside CLAUDE_DIR).

**Step 5: Manual verification — path restriction**

With server running, attempt to read a file outside `~/.claude/`:
```bash
# Should return 400 "Invalid or disallowed path"
curl "http://localhost:3847/api/file-content?path=$(node -e "console.log(require('os').homedir())")/.ssh/known_hosts"
# Expected: {"ok":false,"error":"Invalid or disallowed path"}
```

**Step 6: Commit**

```bash
git add src/server.mjs
git commit -m "security: split path validation into read/write, fix Windows path separator (VULN-003, VULN-009)"
```

---

## Task 3 — Origin Validation / CSRF Protection (VULN-004)

**Files:** Modify `src/server.mjs`

**Step 1: Add `validateOrigin()` function after the `readBody()` helper**

After the closing brace of `readBody()` (around line 86), insert:

```javascript
/**
 * Reject cross-origin POST requests.
 * The browser sets Origin automatically; it cannot be forged by page JavaScript.
 * Requests with no Origin (e.g. curl, server-side fetch) are allowed because
 * this server is localhost-only after VULN-001 fix — curl from LAN is blocked at
 * the network layer.
 */
function isOriginAllowed(req) {
  const origin = req.headers.origin;
  if (!origin) return true; // non-browser client (curl, server-to-server)
  return /^https?:\/\/(localhost|127\.0\.0\.1)(:\d+)?$/.test(origin);
}
```

**Step 2: Add origin check at the top of `handleRequest()` for all mutating methods**

At the very start of `handleRequest()`, after the url/path parsing (around line 104), insert:

```javascript
  // Block cross-origin POST/DELETE/PUT requests (CSRF protection)
  if (req.method !== "GET" && !isOriginAllowed(req)) {
    return json(res, { ok: false, error: "Forbidden" }, 403);
  }
```

**Step 3: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (Playwright makes requests without an Origin header, which `isOriginAllowed` allows).

**Step 4: Manual verification**

```bash
# Simulate a cross-origin POST — should be rejected
curl -X POST http://localhost:3847/api/scan \
  -H "Origin: https://evil.com" \
  -H "Content-Type: application/json" \
  -d '{}'
# Expected: {"ok":false,"error":"Forbidden"}

# Simulate a same-origin POST — should work normally
curl -X POST http://localhost:3847/api/move \
  -H "Origin: http://localhost:3847" \
  -H "Content-Type: application/json" \
  -d '{"itemPath":"/nonexistent","toScopeId":"global","category":"memory","name":"x"}'
# Expected: {"ok":false,"error":"Item not found or locked"} (normal app error, not 403)
```

**Step 5: Commit**

```bash
git add src/server.mjs
git commit -m "security: add Origin header validation to block CSRF attacks (VULN-004)"
```

---

## Task 4 — Request Body Size Limit (VULN-005)

**Files:** Modify `src/server.mjs`

**Step 1: Replace `readBody()` with a size-limited version (lines 78–86)**

Replace:
```javascript
async function readBody(req) {
  let body = "";
  for await (const chunk of req) body += chunk;
  try {
    return JSON.parse(body);
  } catch {
    throw new Error("Invalid JSON body");
  }
}
```
With:
```javascript
async function readBody(req, maxBytes = 1_048_576) { // 1 MB default
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

**Step 2: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (no legitimate test body exceeds 1 MB).

**Step 3: Manual verification**

```bash
# Generate a 2 MB body and POST it — should error gracefully
node -e "process.stdout.write(JSON.stringify({x:'a'.repeat(2*1024*1024)}))" | \
  curl -X POST http://localhost:3847/api/move \
  -H "Content-Type: application/json" \
  --data-binary @-
# Expected: connection closed or {"ok":false,"error":"Internal server error"}
```

**Step 4: Commit**

```bash
git add src/server.mjs
git commit -m "security: add 1 MB request body size limit to prevent OOM DoS (VULN-005)"
```

---

## Task 5 — Restrict Write Endpoints (VULN-002, VULN-006, VULN-013)

**Files:** Modify `src/server.mjs`

Three endpoints can write to attacker-controlled paths. All are fixed by tightening path validation.

### 5a — `/api/restore`: Already uses `isWriteAllowed` from Task 2, which restricts to CLAUDE_DIR. No additional code change needed. VULN-002 is fixed by Task 2.

Verify this is the case: confirm the call site at `/api/restore` now reads `isWriteAllowed(filePath)` and not `isPathAllowed`. If Task 2 was done correctly, this is already fixed.

### 5b — `/api/export`: Restrict export directory to within `~/.claude/` (VULN-006)

Locate `/api/export` (around line 559). Replace the path validation block:
```javascript
// Before
if (!exportDir) exportDir = join(CLAUDE_DIR, "exports");
if (!exportDir.startsWith("/")) {
  return json(res, { ok: false, error: "Invalid exportDir (must be absolute path)" }, 400);
}
```
With:
```javascript
if (!exportDir) exportDir = join(CLAUDE_DIR, "exports");
if (!isAbsolute(exportDir) || !isWriteAllowed(exportDir)) {
  return json(res, { ok: false, error: "exportDir must be within ~/.claude/" }, 400);
}
```

### 5c — `/api/restore-mcp`: Restrict to known `.mcp.json` paths (VULN-013)

Locate `/api/restore-mcp` (around line 447). After `const { name, config, mcpJsonPath } = await readBody(req);`, replace the validation:

```javascript
// Before
if (!name || !config || !mcpJsonPath || !isWriteAllowed(mcpJsonPath)) {
  return json(res, { ok: false, error: "Missing name, config, or mcpJsonPath, or disallowed path" }, 400);
}
```
With:
```javascript
// Build allowlist of valid .mcp.json locations from scanner cache
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

**Note:** If `cachedData` is null (no scan yet), do a fresh scan first: add `if (!cachedData) await freshScan();` before building `knownMcpPaths`.

Full corrected block for 5c:

```javascript
if (path === "/api/restore-mcp" && req.method === "POST") {
  const { name, config, mcpJsonPath } = await readBody(req);
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
  // ... rest of handler unchanged
```

**Step 1: Apply all three changes above**

**Step 2: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (undo-delete for MCP uses real .mcp.json paths the scanner already found).

**Step 3: Manual verification**

```bash
# Test export outside ~/.claude/ — should be rejected
curl -X POST http://localhost:3847/api/export \
  -H "Origin: http://localhost:3847" \
  -H "Content-Type: application/json" \
  -d '{"exportDir":"/tmp/stolen"}'
# Expected: {"ok":false,"error":"exportDir must be within ~/.claude/"}

# Test restore to outside ~/.claude/ — should be rejected
curl -X POST http://localhost:3847/api/restore \
  -H "Origin: http://localhost:3847" \
  -H "Content-Type: application/json" \
  -d '{"filePath":"/tmp/evil.sh","content":"bad","isDir":false}'
# Expected: {"ok":false,"error":"Invalid or disallowed path"}
```

**Step 4: Commit**

```bash
git add src/server.mjs
git commit -m "security: restrict write endpoints to ~/.claude/ only (VULN-002, VULN-006, VULN-013)"
```

---

## Task 6 — Sanitize Error Messages + Scan Rate Limiting (VULN-007, VULN-017)

**Files:** Modify `src/server.mjs`

### 6a — Sanitize error messages (VULN-007)

**Step 1: Fix the global error handler (lines 654–658)**

Replace:
```javascript
    } catch (err) {
      console.error("Error:", err.message);
      res.writeHead(500, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ ok: false, error: err.message }));
    }
```
With:
```javascript
    } catch (err) {
      console.error("Unhandled error:", err);
      res.writeHead(500, { "Content-Type": "application/json", ...SECURITY_HEADERS });
      res.end(JSON.stringify({ ok: false, error: "Internal server error" }));
    }
```

**Step 2: Sanitize inline error returns in `/api/restore` and `/api/restore-mcp`**

In `/api/restore` catch block (around line 441):
```javascript
// Before
return json(res, { ok: false, error: `Restore failed: ${err.message}` }, 400);
// After
console.error("restore error:", err);
return json(res, { ok: false, error: "Restore failed" }, 400);
```

In `/api/restore-mcp` catch block (around line 465):
```javascript
// Before
return json(res, { ok: false, error: `Restore failed: ${err.message}` }, 400);
// After
console.error("restore-mcp error:", err);
return json(res, { ok: false, error: "Restore failed" }, 400);
```

In `/api/export` catch block (around line 620):
```javascript
// Before
return json(res, { ok: false, error: `Export failed: ${err.message}` }, 400);
// After
console.error("export error:", err);
return json(res, { ok: false, error: "Export failed" }, 400);
```

### 6b — Scan rate limiting (VULN-017)

**Step 3: Add cooldown tracking to `freshScan()` (lines 66–69)**

Replace:
```javascript
let cachedData = null;

async function freshScan() {
  cachedData = await scan();
  return cachedData;
}
```
With:
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

Note: mutation endpoints (move, delete, restore) need fresh data after changes, so pass `force = true` at those call sites:

Find all `await freshScan()` calls **inside** mutation handlers (after a successful operation) and change them to `await freshScan(true)`. These are:
- After `moveItem` succeeds (inside `/api/move`, `if (result.ok) await freshScan();`)
- After `deleteItem` succeeds (inside `/api/delete`)
- Inside `/api/restore` after write
- Inside `/api/restore-mcp` after write

The initial `GET /api/scan` call and the `if (!cachedData) await freshScan()` guard calls should stay as `freshScan()` (no force).

**Step 4: Run E2E tests**

```bash
npm test
```
Expected: all tests pass.

**Step 5: Commit**

```bash
git add src/server.mjs
git commit -m "security: sanitize error messages, add scan rate limiting (VULN-007, VULN-017)"
```

---

## Task 7 — Redact MCP Environment Variable Values in Scan Output (VULN-008)

**Files:** Modify `src/scanner.mjs`

MCP server configs in `~/.claude.json` and `.mcp.json` files often contain API keys in their `env` field. The scanner exposes these via `/api/scan`. This task redacts the values (but preserves the keys so users can see what variables are configured).

**Step 1: Add a `redactMcpConfig()` helper near the top of scanner.mjs** (after imports, before any scan functions)

```javascript
/**
 * Return a copy of an MCP server config with environment variable values redacted.
 * Keys are preserved so users can see which variables are configured.
 */
function redactMcpConfig(serverConfig) {
  if (!serverConfig || typeof serverConfig !== "object") return serverConfig;
  if (!serverConfig.env || typeof serverConfig.env !== "object") return serverConfig;
  return {
    ...serverConfig,
    env: Object.fromEntries(
      Object.entries(serverConfig.env).map(([k, v]) => [k, typeof v === "string" ? "●●●●●●●●" : v])
    ),
  };
}
```

**Step 2: Apply `redactMcpConfig()` at every `mcpConfig:` assignment in `scanMcp()`**

There are four locations. Find each `mcpConfig: serverConfig,` and replace with `mcpConfig: redactMcpConfig(serverConfig),`:

- Line ~503 (`.claude.json` user-scope loop)
- Line ~527 (`.claude.json` project-scope loop)
- Line ~557 (`.mcp.json` file loop)
- Line ~592 (settings files loop)

**Step 3: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (E2E tests check item metadata like name/category, not env var values).

**Step 4: Manual verification**

If you have any MCP server configured with an `env` field, run:
```bash
curl http://localhost:3847/api/scan | \
  node -e "const d=require('fs').readFileSync('/dev/stdin','utf8'); \
  const items=JSON.parse(d).items.filter(i=>i.category==='mcp'); \
  items.forEach(i=>console.log(i.name, JSON.stringify(i.mcpConfig?.env)));"
# Expected: env values show as "●●●●●●●●", not actual API keys
```

**Step 5: Commit**

```bash
git add src/scanner.mjs
git commit -m "security: redact MCP env var values from scan output (VULN-008)"
```

---

## Task 8 — Remove existsSync TOCTOU + Prototype Pollution in mover.mjs (VULN-012, VULN-016)

**Files:** Modify `src/mover.mjs`

### 8a — Replace blocking `existsSync` with async `access` (VULN-012)

**Step 1: Update the import at line 13 to add `access` from `node:fs/promises` and remove `existsSync` from `node:fs`**

Replace:
```javascript
import { rename, mkdir, readFile, writeFile, rm, unlink, cp } from "node:fs/promises";
```
With:
```javascript
import { rename, mkdir, readFile, writeFile, rm, unlink, cp, access } from "node:fs/promises";
```

Remove or update the synchronous import:
```javascript
import { existsSync } from "node:fs";
```
→ Delete this line entirely (we no longer need it).

**Step 2: Replace all six `existsSync(toPath)` guard patterns**

Each move function has this pattern:
```javascript
if (existsSync(toPath)) {
  return { ok: false, error: `File already exists at destination: ...` };
}
```

Replace every occurrence with an async equivalent:
```javascript
try { await access(toPath); return { ok: false, error: `File already exists at destination: ...` }; } catch {}
```

There are 6 occurrences at lines ~114, ~139, ~160, ~184, ~209, ~233. Apply to all 6.

Example for `moveMemory` (line 114):
```javascript
// Before
if (existsSync(toPath)) {
  return { ok: false, error: `File already exists at destination: ${item.fileName}` };
}
// After
try { await access(toPath); return { ok: false, error: `File already exists at destination: ${item.fileName}` }; } catch {}
```

### 8b — Prototype pollution protection (VULN-016)

**Step 3: Add `sanitizeKeys()` helper at the top of mover.mjs** (after imports)

```javascript
const DANGEROUS_KEYS = new Set(["__proto__", "constructor", "prototype"]);

/** Strip prototype-polluting keys from a parsed JSON object's top-level keys. */
function sanitizeKeys(obj) {
  if (!obj || typeof obj !== "object") return obj;
  return Object.fromEntries(Object.entries(obj).filter(([k]) => !DANGEROUS_KEYS.has(k)));
}
```

**Step 4: Apply `sanitizeKeys()` around `mcpServers` access in `moveMcp()` and `deleteMcp()`**

In `moveMcp()` (around line 265):
```javascript
// Before
const serverConfig = fromContent.mcpServers?.[item.name];
// After
const servers = sanitizeKeys(fromContent.mcpServers || {});
const serverConfig = servers[item.name];
```

In `moveMcp()` destination check (around line 279):
```javascript
// Before
if (toContent.mcpServers[item.name]) {
// After
const toServers = sanitizeKeys(toContent.mcpServers || {});
if (toServers[item.name]) {
```
And when adding to destination:
```javascript
// Before
toContent.mcpServers[item.name] = serverConfig;
// After
if (!toContent.mcpServers) toContent.mcpServers = {};
toContent.mcpServers[item.name] = serverConfig;
```

In `deleteMcp()` (around line 361):
```javascript
// Before
if (!content.mcpServers?.[item.name]) {
// After
const servers = sanitizeKeys(content.mcpServers || {});
if (!servers[item.name]) {
```

**Step 5: Run E2E tests**

```bash
npm test
```
Expected: all tests pass.

**Step 6: Commit**

```bash
git add src/mover.mjs
git commit -m "security: replace existsSync TOCTOU with async access, add prototype pollution guard (VULN-012, VULN-016)"
```

---

## Task 9 — Prototype Pollution in scanner.mjs (VULN-016)

**Files:** Modify `src/scanner.mjs`

**Step 1: Add the same `DANGEROUS_KEYS` / `sanitizeKeys` helper to scanner.mjs** (after imports)

```javascript
const DANGEROUS_KEYS = new Set(["__proto__", "constructor", "prototype"]);

function sanitizeKeys(obj) {
  if (!obj || typeof obj !== "object") return obj;
  return Object.fromEntries(Object.entries(obj).filter(([k]) => !DANGEROUS_KEYS.has(k)));
}
```

**Step 2: Wrap `mcpServers` access in all four MCP-scanning loops in `scanMcp()`**

At each location where `Object.entries(claudeJson.mcpServers)`, `Object.entries(projMcp)`, `Object.entries(servers)` is called, wrap the source in `sanitizeKeys()`:

```javascript
// Before (example at line 486)
for (const [name, serverConfig] of Object.entries(claudeJson.mcpServers)) {
// After
for (const [name, serverConfig] of Object.entries(sanitizeKeys(claudeJson.mcpServers))) {
```

Apply to all 4 `Object.entries(...)` calls inside `scanMcp()` that iterate MCP server maps.

**Step 3: Run E2E tests**

```bash
npm test
```
Expected: all tests pass.

**Step 4: Commit**

```bash
git add src/scanner.mjs
git commit -m "security: add prototype pollution guard to MCP config parsing in scanner (VULN-016)"
```

---

## Task 10 — Browser Launch: Replace execSync with execFile, Add Windows Support (VULN-015)

**Files:** Modify `bin/cli.mjs`

**Step 1: Replace the child_process import and browser-open block (lines 89, 108–113)**

Replace:
```javascript
  const { execSync } = await import('node:child_process');
```
With:
```javascript
  const { execFile } = await import('node:child_process');
```

Replace the `try` block (lines 108–113):
```javascript
  try {
    const openCmd = process.platform === 'darwin' ? 'open' : 'xdg-open';
    execSync(`${openCmd} http://localhost:${port}`, { stdio: 'ignore' });
  } catch {
    // Browser didn't open, user can navigate manually
  }
```
With:
```javascript
  try {
    const openCmds = { darwin: 'open', win32: 'cmd', linux: 'xdg-open' };
    const platform = process.platform === 'win32' ? 'win32'
                   : process.platform === 'darwin' ? 'darwin' : 'linux';
    const openCmd = openCmds[platform];
    const args = platform === 'win32'
      ? ['/c', 'start', '', `http://localhost:${port}`]
      : [`http://localhost:${port}`];
    execFile(openCmd, args, { stdio: 'ignore' }, () => {});
  } catch {
    // Browser didn't open — user can navigate manually
  }
```

**Step 2: Run E2E tests**

```bash
npm test
```
Expected: all tests pass (E2E tests don't exercise the browser-open path).

**Step 3: Manual verification**

Run `npm start` and confirm the browser opens automatically on your platform.

**Step 4: Commit**

```bash
git add bin/cli.mjs
git commit -m "security: replace execSync with execFile for browser launch, add Windows support (VULN-015)"
```

---

## Task 11 — Frontend: SRI Hash + System Fonts (VULN-011, VULN-014)

**Files:** Modify `src/ui/index.html`

The SRI hash for `sortablejs@1.15.6` (computed during planning):
`sha384-HZZ/fukV+9G8gwTNjN7zQDG0Sp7MsZy5DDN6VfY3Be7V9dvQpEpR2jF2HlyFUUjU`

**Step 1: Add SRI hash to the SortableJS script tag (line 9)**

Replace:
```html
  <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"></script>
```
With:
```html
  <script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"
    integrity="sha384-HZZ/fukV+9G8gwTNjN7zQDG0Sp7MsZy5DDN6VfY3Be7V9dvQpEpR2jF2HlyFUUjU"
    crossorigin="anonymous"></script>
```

**Step 2: Replace Google Fonts with system fonts (line 8)**

Remove:
```html
  <link href="https://fonts.googleapis.com/css2?family=Lato:wght@400;700;900&family=Geist+Mono:wght@400;500&display=swap" rel="stylesheet">
```

Then open `src/ui/style.css` and find the font-family declarations that reference `Lato` and `Geist Mono`. Replace them with system font stacks:

For `Lato` usages — replace with:
```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
```

For `Geist Mono` / monospace usages — replace with:
```css
font-family: ui-monospace, "Cascadia Code", "Fira Code", "Consolas", "Menlo", monospace;
```

Run a search in `style.css` to find all occurrences:
```bash
grep -n "Lato\|Geist" src/ui/style.css
```
Apply replacements at every hit.

Also update the CSP `font-src` in `SECURITY_HEADERS` (from Task 1) to remove the Google entries since we no longer load from there:

In `server.mjs`, replace:
```javascript
  "font-src 'self' https://fonts.gstatic.com data:",
```
With:
```javascript
  "font-src 'self' data:",
```

And remove `https://fonts.googleapis.com` from `style-src`:
```javascript
// Before
  "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
// After
  "style-src 'self' 'unsafe-inline'",
```

**Step 3: Run E2E tests**

```bash
npm test
```
Expected: all tests pass.

**Step 4: Manual verification**

Open the dashboard, confirm it loads with system fonts and that no requests appear to `fonts.googleapis.com` or `fonts.gstatic.com` in the browser Network tab. Confirm drag-and-drop still works (SortableJS loaded correctly).

**Step 5: Commit**

```bash
git add src/ui/index.html src/ui/style.css src/server.mjs
git commit -m "security: add SRI hash to SortableJS CDN, replace Google Fonts with system fonts (VULN-011, VULN-014)"
```

---

## Task 12 — Fix Frontend fetchJson to Check HTTP Status (VULN-018)

**Files:** Modify `src/ui/app.js`

**Step 1: Find the `fetchJson` function**

Search for it:
```bash
grep -n "fetchJson\|async function fetch\|function fetch" src/ui/app.js | head -20
```

**Step 2: Update `fetchJson` to check `res.ok`**

The function currently looks like:
```javascript
async function fetchJson(url) {
  const res = await fetch(url);
  return res.json();
}
```

Replace with:
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

**Step 3: Run E2E tests**

```bash
npm test
```
Expected: all tests pass.

**Step 4: Commit**

```bash
git add src/ui/app.js
git commit -m "security: check res.ok in fetchJson to surface HTTP errors (VULN-018)"
```

---

## Task 13 — Final Verification Pass

**Step 1: Run full E2E suite one final time**

```bash
npm test
```
Expected: all tests pass.

**Step 2: Run the server and do a complete manual walkthrough**

```bash
npm start
```

Checklist:
- [ ] Dashboard loads at `http://localhost:3847`
- [ ] Browser Network tab shows `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` on responses
- [ ] Server only listens on 127.0.0.1 (check with `netstat -an | grep 3847`)
- [ ] No requests to `fonts.googleapis.com` in Network tab
- [ ] Drag-and-drop works (SortableJS with SRI loaded)
- [ ] Move an item between scopes → works
- [ ] Delete an item → works
- [ ] Undo delete → works
- [ ] Export to default location (`~/.claude/exports/`) → works
- [ ] `GET /api/scan` response: MCP env values show as `●●●●●●●●` (if you have MCP servers with env vars)
- [ ] `GET /api/file-content?path=/etc/passwd` → 400 (blocked outside CLAUDE_DIR/repo dirs)
- [ ] `POST /api/restore` with path outside `~/.claude/` → 400

**Step 3: Final commit with version note in SECURITY-AUDIT.md**

Add a one-line status at the top of SECURITY-AUDIT.md:
```markdown
> **Status:** All 18 findings remediated in v0.9.2 — see git log for details.
```

```bash
git add SECURITY-AUDIT.md
git commit -m "docs: mark all OWASP findings remediated"
```

---

## Commit Log Summary (expected)

After all tasks, `git log --oneline` should show:
```
docs: mark all OWASP findings remediated
security: check res.ok in fetchJson to surface HTTP errors (VULN-018)
security: add SRI hash to SortableJS CDN, replace Google Fonts with system fonts (VULN-011, VULN-014)
security: replace execSync with execFile for browser launch, add Windows support (VULN-015)
security: add prototype pollution guard to MCP config parsing in scanner (VULN-016)
security: replace existsSync TOCTOU with async access, add prototype pollution guard (VULN-012, VULN-016)
security: redact MCP env var values from scan output (VULN-008)
security: sanitize error messages, add scan rate limiting (VULN-007, VULN-017)
security: restrict write endpoints to ~/.claude/ only (VULN-002, VULN-006, VULN-013)
security: add 1 MB request body size limit to prevent OOM DoS (VULN-005)
security: add Origin header validation to block CSRF attacks (VULN-004)
security: split path validation into read/write, fix Windows path separator (VULN-003, VULN-009)
security: bind server to 127.0.0.1 and add security headers (VULN-001, VULN-010)
```

---

## Vulnerability Coverage Matrix

| VULN | Description | Task |
|------|-------------|------|
| VULN-001 | Bind to 127.0.0.1 | Task 1 |
| VULN-002 | Restrict /api/restore writes | Task 2 (isWriteAllowed) + Task 5 |
| VULN-003 | Split path allowlist read/write | Task 2 |
| VULN-004 | CSRF / Origin validation | Task 3 |
| VULN-005 | Body size limit | Task 4 |
| VULN-006 | Export path restriction | Task 5b |
| VULN-007 | Error message sanitization | Task 6a |
| VULN-008 | MCP env var redaction | Task 7 |
| VULN-009 | Windows path separator fix | Task 2 |
| VULN-010 | Security headers | Task 1 |
| VULN-011 | SortableJS SRI hash | Task 11 |
| VULN-012 | TOCTOU existsSync → async | Task 8 |
| VULN-013 | restore-mcp path allowlist | Task 5c |
| VULN-014 | System fonts (no Google) | Task 11 |
| VULN-015 | execFile + Windows browser open | Task 10 |
| VULN-016 | Prototype pollution guard | Tasks 8+9 |
| VULN-017 | Scan rate limiting | Task 6b |
| VULN-018 | fetchJson res.ok check | Task 12 |
