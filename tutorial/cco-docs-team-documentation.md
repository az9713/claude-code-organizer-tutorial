# Agent Team Documentation
## Project: CCO Technical Documentation — `cco-docs-team`
### A Complete Technical Account of a Multi-Agent Documentation Production System

---

> **Purpose of this document:** To fully demystify how a team of AI agents collaborated to research, write, edit, and assemble a comprehensive technical document about the claude-code-organizer codebase. Every agent, every turn, every message, every decision is documented here — nothing is left as a black box.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [The Agents — Full Profiles](#3-the-agents--full-profiles)
   - 3.1 The Orchestrator (Main Claude Session)
   - 3.2 The Researcher
   - 3.3 The Writer
   - 3.4 The Editor
4. [The Task System](#4-the-task-system)
5. [The Communication Protocol](#5-the-communication-protocol)
6. [Complete Turn-by-Turn Narrative](#6-complete-turn-by-turn-narrative)
   - Phase 1: Team Architecture
   - Phase 2: Research
   - Phase 3: The Write-Edit Loop (9 sections)
   - Phase 4: Assembly
   - Phase 5: Shutdown
7. [All Artifacts Produced](#7-all-artifacts-produced)
8. [Inter-Agent Message Log](#8-inter-agent-message-log)
9. [Quality Metrics — The Write-Edit Loop](#9-quality-metrics--the-write-edit-loop)
10. [Key Design Principles Illustrated](#10-key-design-principles-illustrated)
11. [What Could Go Wrong — Edge Cases Encountered](#11-what-could-go-wrong--edge-cases-encountered)
12. [Summary Statistics](#12-summary-statistics)

---

## 1. Project Overview

### Goal

Produce a comprehensive technical document explaining how the claude-code-organizer (CCO) works internally — its algorithms, data structures, and design decisions — suitable for a developer who wants to understand the codebase without reading it line by line.

### The Team

A coordinated team of **3 named AI agents** plus a **human-facing orchestrator** (the main Claude session).

### The Result

- **File:** `docs/how-it-works.md`
- **Sections:** 9
- **Approximate length:** ~4,500 words
- **Structure:** Overview → Scope Discovery → Scanner → Mover → Context Budget → Dashboard UI → MCP Server → Plugin/Skill System → Technologies
- **Time:** ~45 minutes end-to-end

### Source Materials Read

| File | Lines | Content |
|---|---|---|
| `src/scanner.mjs` | ~1,013 | Core scope discovery + 11-category scanner |
| `src/mover.mjs` | ~445 | Move/delete items between scopes |
| `src/server.mjs` | ~689 | HTTP server, REST API, context budget |
| `src/mcp-server.mjs` | ~135 | MCP server with 4 tools |
| `src/tokenizer.mjs` | ~59 | Token counting |
| `src/ui/app.js` | ~2,245 | Dashboard frontend |
| `src/ui/index.html` | ~191 | Three-panel layout |
| `src/ui/style.css` | ~1,012 | Styling |
| `bin/cli.mjs` | ~114 | Entry point + pre-flight + auto-install |
| `package.json` | — | Dependencies, scripts, packaging config |
| `.claude-plugin/` | 3 files | Plugin packaging |

**Total source lines read by researcher:** ~6,000+

---

## 2. System Architecture

### Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR (Team Lead)                      │
│  • Designs team structure       • Monitors idle_notifications   │
│  • Creates tasks & dependencies  • Intervenes when stuck        │
│  • Spawns all agents            • Assembles final document      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ spawns + manages
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │RESEARCHER│      │  WRITER  │      │  EDITOR  │
    │ Task #1  │      │ Task #2  │      │ Task #3  │
    └────┬─────┘      └────┬─────┘      └────┬─────┘
         │                 │  ◄────────────►  │
   docs_research_    docs_draft_section_N.md  │
   notes.md         docs_final_section_N.md  │
         │                 │                  │
         └─────────────────┴──────────────────┘
                           │ all complete
                           ▼
                  ┌──────────────────┐
                  │   ORCHESTRATOR   │
                  │   Task #4        │
                  │  (assembles      │
                  │  final doc)      │
                  └────────┬─────────┘
                           │
                  docs/how-it-works.md
```

### Task Dependency Graph

```
Task #1: Research
    ├── blocks Task #2 (Write)
    └── blocks Task #3 (Edit)

Task #2: Write  ◄──► Task #3: Edit   (iterative loop, 9 sections)
    └── both block Task #4 (Assembly)

Task #4: Assemble Final Document
    └── terminal node — no downstream dependencies
```

### Agent Communication Topology

```
Orchestrator ──SendMessage──► Researcher  (shutdown only)
Orchestrator ──SendMessage──► Writer      (shutdown only)
Orchestrator ──SendMessage──► Editor      (shutdown only)
Researcher   ──SendMessage──► Writer      (handoff: research ready)
Writer       ──SendMessage──► Editor      (draft ready for critique) ×9
Editor       ──SendMessage──► Writer      (critique / approval) ×9×2
Writer       ──SendMessage──► Orchestrator (all 9 sections done)
Editor       ──SendMessage──► Orchestrator (all 9 sections approved)
```

**Key insight:** The orchestrator did **not** need to send go-ahead messages to the writer or editor — the task dependency system handled that automatically. Unlike the Munger article team (which needed orchestrator intervention to bridge a pre-team background agent), this team ran entirely autonomously from research through approval with zero orchestrator interventions.

---

## 3. The Agents — Full Profiles

---

### 3.1 The Orchestrator — Main Claude Session

| Property | Detail |
|---|---|
| **Identity** | The main Claude instance the user talks to directly |
| **Team role** | Team architect, task manager, assembler, shutdown controller |
| **Agent type** | N/A — this is the root session, not a spawned agent |
| **Tools available** | All tools: `TeamCreate`, `TaskCreate`, `TaskUpdate`, `TaskList`, `Agent`, `SendMessage`, `Read`, `Write`, `Glob`, `Grep`, etc. |

#### What the Orchestrator Does

The orchestrator designed and launched the team, then stepped back and let the agents run. Its contributions:

1. **Explored the codebase** before spawning any agents — using an Explore subagent to read all source files and understand the architecture. This informed the agent prompts.
2. **Read the workflow template** — `.ignore/agent_team_documentation.md` — to understand the proven team pattern before designing the new team.
3. **Created the team namespace** via `TeamCreate("cco-docs-team")`.
4. **Created all 4 tasks** with descriptions, active forms, and dependency chains.
5. **Spawned all 3 agents simultaneously** with detailed role prompts (~600–900 words each).
6. **Monitored `idle_notification` events** — passively received 40+ agent state updates without polling.
7. **Made zero interventions** during the write-edit loop — the pipeline ran cleanly end-to-end.
8. **Assembled Task #4 itself** — read all 9 final section files and wrote the assembled `docs/how-it-works.md`.
9. **Issued shutdown requests** to all agents at the end.

#### Why the Orchestrator Handled Assembly Itself

Unlike the Munger article team (which spawned an ad-hoc `builder` subagent for PDF assembly), the orchestrator handled Task #4 directly. Reason: the assembly task — read 9 markdown files, combine them with a table of contents — was straightforward file I/O that the orchestrator could do in one pass. Spawning a specialist subagent would have added overhead without benefit.

#### Who the Orchestrator Talks To
- All agents via `SendMessage` (shutdown phase only)
- The user directly (text output)

---

### 3.2 The Researcher (`researcher@cco-docs-team`)

| Property | Detail |
|---|---|
| **Name** | `researcher` |
| **Team ID** | `researcher@cco-docs-team` |
| **Persona** | Deep codebase analyst — reads source code thoroughly and produces structured reference notes |
| **Agent type** | General-purpose (all tools) |
| **Assigned task** | Task #1 |
| **Task status at spawn** | Immediately claimable (no blockers) |

#### What the Researcher Does

The researcher is the **pipeline's first node**. Nothing moves forward until it completes. Its job: read ~6,000 lines of source code and produce a structured reference document that writers can use without re-reading the code themselves.

**Step 1 — Discovers teammates**
Reads `~/.claude/teams/cco-docs-team/config.json` to learn the names of other agents.

**Step 2 — Claims Task #1**
Calls `TaskList`, finds Task #1 unblocked and unowned. Claims it: `TaskUpdate(taskId="1", owner="researcher", status="in_progress")`.

**Step 3 — Reads all source files in chunks**
Large files are read with `offset` and `limit` parameters (the `Read` tool supports this):
- `src/scanner.mjs` (~1,013 lines): read in 5 chunks of ~200 lines
- `src/ui/app.js` (~2,245 lines): read in ~9-10 chunks
- All other files read in 1-2 passes

**Step 4 — Writes `docs_research_notes.md`**
Produces a structured document at `.ignore/docs_research_notes.md` covering 9 parts (A through I):
- **Part A**: Project Overview & Architecture
- **Part B**: Scope Discovery Engine (algorithm, path decoding, hierarchy building)
- **Part C**: The 11-Category Scanner (all 11 scanners documented individually)
- **Part D**: The Mover System (category-to-path mapping, validation, safeRename)
- **Part E**: Context Budget Calculator (always-loaded vs deferred, token counting)
- **Part F**: Dashboard UI (REST API routes, drag-and-drop, undo system)
- **Part G**: MCP Server Mode (4 tools, caching strategy)
- **Part H**: Plugin & Skill System (.claude-plugin/ structure, auto-install)
- **Part I**: Technologies & Dependencies (runtime, deps, CDN libs, CI/CD)

**Step 5 — Notifies writer and closes task**
```
SendMessage(to="writer", summary="Research notes complete -- ready at docs_research_notes.md")
TaskUpdate(taskId="1", status="completed")
```

#### Inputs
- 12 source files (read via `Read` with offset/limit)
- No web searches required (unlike the Munger team) — codebase was the authoritative source

#### Output
- `.ignore/docs_research_notes.md` — comprehensive reference notes

#### Who the Researcher Talks To
| Direction | Recipient | Content |
|---|---|---|
| Receives | Orchestrator | Initial spawn prompt |
| Sends | Writer | "Research notes complete at docs_research_notes.md" |
| Receives | Orchestrator | Shutdown request |
| Sends | Orchestrator | `shutdown_approved` |

---

### 3.3 The Writer (`writer@cco-docs-team`)

| Property | Detail |
|---|---|
| **Name** | `writer` |
| **Team ID** | `writer@cco-docs-team` |
| **Persona** | Technical documentation author — writes for a developer audience, concrete and specific |
| **Agent type** | General-purpose (all tools) |
| **Assigned task** | Task #2 |
| **Task status at spawn** | Blocked by Task #1 |

#### What the Writer Does

The writer is the **central production node**. It works in a strict sequential loop, processing one section at a time and never advancing until the previous section is editor-approved.

**Preparation Phase:**
When spawned with Task #2 still blocked, the writer calls `TaskList`, sees the blocked state, and immediately goes idle. It does not poll or retry — it simply waits for the idle_notification system to deliver the researcher's message when Task #1 completes.

**The Per-Section Loop (executed 9 times):**

```
For each section N (1 through 9):

  1. Read docs_research_notes.md (parts relevant to section N)
  2. Draft section prose (400-700 words)
  3. Write to docs_draft_section_N.md via Write tool
  4. SendMessage to editor: "Section N draft ready — please read docs_draft_section_N.md"
  5. Go idle (await editor critique)

  [Editor sends critique via SendMessage]

  6. Wake from idle, read critique
  7. Revise draft addressing all specific points
  8. Write to docs_final_section_N.md via Write tool
  9. SendMessage to editor: "Revised Section N ready — please read docs_final_section_N.md"
  10. Go idle (await approval)

  [Editor sends approval via SendMessage]

  11. Wake from idle, receive approval
  12. Repeat for Section N+1
```

**The 9 Sections Written:**

| N | Title | Core Content |
|---|---|---|
| 1 | Overview & Architecture | Purpose, dual-mode design, zero-build, project structure |
| 2 | Scope Discovery Engine | Path encoding, greedy decoding algorithm, hierarchy building |
| 3 | The 11-Category Scanner | All 11 scanners with filesystem locations, parse logic |
| 4 | The Mover System | Category-to-path mapping, validateMove(), safeRename(), MCP JSON moves |
| 5 | Context Budget Calculator | Always-loaded vs deferred, scope chain, token counting |
| 6 | The Dashboard UI | Three-panel layout, REST API, drag-and-drop, undo system |
| 7 | MCP Server Mode | 4 tools, Zod validation, caching strategy |
| 8 | Plugin & Skill System | .claude-plugin/ structure, auto-install, skill format |
| 9 | Technologies & Dependencies | Stack, CI/CD pipeline, zero-build rationale |

#### Writing Standards Applied

The writer was prompted to apply these standards to every section:
- **Developer audience** — readers want to understand the codebase, not use the tool
- **Every abstract claim backed by a concrete example** — function name, file path, line number range
- **Use code blocks** for algorithms and data structures
- **Strong opening sentence** that states the section's core concept immediately
- **Brief closing** explaining why the design choice matters

#### Inputs
- `.ignore/docs_research_notes.md` (via `Read`)
- Editor critiques (via `SendMessage`) — one or two per section

#### Outputs
- `docs_draft_section_1.md` through `docs_draft_section_9.md` (first drafts)
- `docs_final_section_1.md` through `docs_final_section_9.md` (revised finals)

#### Who the Writer Talks To
| Direction | Recipient | Content |
|---|---|---|
| Receives | Researcher | "Research notes complete — begin writing" |
| Sends | Editor | "Section N draft ready" (×9) |
| Receives | Editor | Critiques (×9) |
| Sends | Editor | "Revised Section N ready" (×9) |
| Receives | Editor | "Section N approved — proceed to N+1" (×9) |
| Sends | Orchestrator | "All 9 sections written and approved" |
| Receives | Orchestrator | Shutdown request |
| Sends | Orchestrator | `shutdown_approved` |

---

### 3.4 The Editor (`editor@cco-docs-team`)

| Property | Detail |
|---|---|
| **Name** | `editor` |
| **Team ID** | `editor@cco-docs-team` |
| **Persona** | Demanding, exacting technical editor — accuracy and specificity over vague description |
| **Agent type** | General-purpose (all tools) |
| **Assigned task** | Task #3 |
| **Task status at spawn** | Blocked by Task #1 |

#### What the Editor Does

The editor is the **quality control node** — the gatekeeper ensuring every section is technically accurate before it enters the final document. Every section must pass through the editor before the writer can advance.

**Preparation Phase:**
When Task #1 completes, the editor proactively reads:
- `.ignore/docs_research_notes.md` — to calibrate accuracy checks
- Key source files (`scanner.mjs`, `mover.mjs`) — to verify claims against the actual code

This preparation allowed the editor to catch inaccuracies that would otherwise have required re-reading source files mid-critique.

**The Per-Section Critique Loop:**

```
For each section N:

  1. Receive SendMessage from writer: "Section N draft ready"
  2. Read docs_draft_section_N.md
  3. Cross-check against docs_research_notes.md and source code
  4. Apply 5-criterion evaluation framework
  5. Write specific, actionable critique
  6. SendMessage to writer with full critique
  7. Go idle (await revision)

  [Writer sends revision]

  8. Read docs_final_section_N.md
  9. Verify critique was addressed
  10. SendMessage to writer: "Section N approved — proceed to N+1"
  11. Go idle
```

#### The 5-Criterion Evaluation Framework

Every section was evaluated on all 5 criteria:

**1. Technical Accuracy**
- Are function names, file paths, and algorithms correct?
- Do they match what's actually in the source code?
- Verified by cross-referencing docs_research_notes.md and the source files directly

**2. Completeness**
- Is anything important missing?
- Are key design decisions explained, not just described?

**3. Clarity**
- Would a developer unfamiliar with the codebase understand it?
- Does it build understanding progressively?

**4. Specificity**
- Are concrete names used (functions, variables, line numbers)?
- Is vague language eliminated?

**5. Flow**
- Is there a clear through-line?
- Does each paragraph lead to the next?

#### Critique Style — Specific, Not Vague

The editor was explicitly instructed to name what was wrong and say what should replace it:

- ❌ **Vague:** "The second paragraph could be more precise."
- ✅ **Specific:** "The second paragraph says the safeRename function moves files 'atomically' — this is inaccurate. The function writes destination first, then removes source, which is sequential not atomic. Replace with: 'writes destination first, then removes source — safe but not atomic.'"

#### What the Editor Caught (from its own summary)

7 categories of issues corrected across 9 sections:
1. **Incorrect category name** — "settings" used where the correct term is "configs"
2. **Missing features** — pre-flight check and `/cco` auto-install omitted from Section 1
3. **Code snippet inaccuracies** — `safeRename()` `isDir` parameter described incorrectly; MCP server hardcoded version `"0.5.0"` not mentioned
4. **Misleading terminology** — "atomically" used for sequential write-then-delete
5. **JSON response shape mismatches** — field names and data types in the context budget API response were wrong
6. **Fabricated detail** — `--experimental-vm-modules` flag included but does not exist in the codebase
7. **Misquoted skill description** — the `/cco` skill description text was paraphrased incorrectly

#### Inputs
- `.ignore/docs_research_notes.md` (read proactively)
- `src/scanner.mjs`, `src/mover.mjs` (read for cross-verification)
- `docs_draft_section_N.md` (×9, as drafts arrive)
- `docs_final_section_N.md` (×9, after revisions)

#### Outputs
- Critique messages to writer (×9 via `SendMessage`)
- Approval messages to writer (×9 via `SendMessage`)

#### Who the Editor Talks To
| Direction | Recipient | Content |
|---|---|---|
| Receives | Writer | "Section N draft ready" (×9) |
| Sends | Writer | Full critiques (×9) |
| Sends | Writer | "Section N approved — proceed to N+1" (×9) |
| Sends | Orchestrator | "All 9 sections reviewed and approved" |
| Receives | Orchestrator | Shutdown request |
| Sends | Orchestrator | `shutdown_approved` |

---

## 4. The Task System

### What Tasks Are

Tasks are **shared state objects** stored in `~/.claude/tasks/cco-docs-team/`. Every agent on the team can read and update them. They serve three purposes simultaneously:
1. **Work queue** — agents check for unowned, unblocked tasks to claim
2. **Progress tracker** — `status` field (`pending` / `in_progress` / `completed`) shows pipeline state
3. **Dependency enforcer** — `blockedBy` prevents agents from starting work they shouldn't do yet

### Task Definitions

**Task #1 — Research**
```
Subject:     Research: Deep-read all source files and document every technique
Status flow: pending → in_progress (researcher) → completed (researcher)
Blockers:    None
Blocks:      Tasks #2 and #3
Owner:       researcher
```

**Task #2 — Write Documentation**
```
Subject:     Write: Draft all 9 documentation sections
Status flow: pending [blocked] → in_progress (writer) → completed (writer)
Blockers:    Task #1
Blocks:      Task #4
Owner:       writer
```

**Task #3 — Edit & Critique**
```
Subject:     Edit: Critique each documentation section for accuracy and quality
Status flow: pending [blocked] → in_progress (editor) → completed (editor)
Blockers:    Task #1
Blocks:      Task #4
Owner:       editor
```

**Task #4 — Assemble Final Document**
```
Subject:     Assemble: Combine all approved sections into docs/how-it-works.md
Status flow: pending [blocked] → in_progress (orchestrator) → completed (orchestrator)
Blockers:    Tasks #2 AND #3
Blocks:      Nothing (terminal)
Owner:       orchestrator (team-lead)
```

### The Dependency System in Practice

When Task #1 was `in_progress`, `TaskList` showed:
```
#1 [in_progress] Research (researcher)
#2 [pending] Write — blocked by #1
#3 [pending] Edit — blocked by #1
#4 [pending] Assemble — blocked by #2, #3
```

Writer and editor saw this and understood there was nothing to do. No message from the orchestrator was needed — the task system handled idle state automatically.

When Task #1 was marked `completed` by the researcher, Tasks #2 and #3 became unblocked. The orchestrator also received the researcher's `idle_notification` with the summary "[to writer] Research notes complete", which confirmed the handoff had happened.

### Who Updated Which Tasks

| Task | Event | Updated by | Method |
|---|---|---|---|
| #1 | Claimed | Researcher | `TaskUpdate(owner=researcher, status=in_progress)` |
| #1 | Completed | Researcher | `TaskUpdate(status=completed)` — self-reported |
| #2 | Claimed | Writer | `TaskUpdate(owner=writer, status=in_progress)` |
| #2 | Completed | Writer | `TaskUpdate(status=completed)` — self-reported |
| #3 | Claimed | Editor | `TaskUpdate(owner=editor, status=in_progress)` |
| #3 | Completed | Editor | `TaskUpdate(status=completed)` — self-reported |
| #4 | Claimed | Orchestrator | `TaskUpdate(owner=team-lead, status=in_progress)` |
| #4 | Completed | Orchestrator | `TaskUpdate(status=completed)` — after assembling file |

**Notable difference from Munger team:** On the CCO docs team, all 4 tasks completed cleanly by their natural owners. The Munger team required the orchestrator to manually close Task #1 and re-route Task #4 to a different agent. This team's cleaner flow came from better upfront planning: task descriptions were detailed enough that agents knew exactly what to do.

---

## 5. The Communication Protocol

### The `SendMessage` Tool

`SendMessage` is the **only channel** through which agents communicate with each other. Key properties:
- Addressed by **name** (`to: "writer"`, `to: "editor"`) — never by ID
- Includes a `summary` field (5–10 words) shown in the orchestrator's UI as a preview
- Delivery is **asynchronous** — the sender goes idle; the recipient wakes when the message arrives
- Messages from teammates are **automatically delivered** — the orchestrator does not need to check an inbox

### The `idle_notification` Signal

Every time an agent completes a processing turn, the system sends an `idle_notification` to the orchestrator automatically. This is how the orchestrator tracked pipeline health without polling.

The notification has two forms:

**Simple idle (agent waiting, nothing to report):**
```json
{"type": "idle_notification", "from": "writer", "timestamp": "...", "idleReason": "available"}
```

**Idle with peer message summary (agent sent a message to another agent):**
```json
{
  "type": "idle_notification",
  "from": "writer",
  "idleReason": "available",
  "summary": "[to editor] Section 3 draft ready for review"
}
```

The `summary` field gave the orchestrator **visibility into peer-to-peer communication** without seeing the full message content. This is how the orchestrator tracked all 18 write-edit sub-turns without the writer or editor explicitly reporting to it.

### Idle Notification Interpretation Guide

| Who | Summary / State | Meaning | Orchestrator action |
|---|---|---|---|
| writer | idle, no summary | Waiting for Task #1 | None — expected blocked state |
| researcher | `[to writer] Research notes complete` | Handoff sent | Confirmed Task #1 done |
| writer | `[to editor] Section N draft ready` | Draft sent, awaiting critique | Noted progress |
| writer | `[to editor] Revised Section N ready` | Revision sent, awaiting approval | Noted progress |
| editor | idle (no summary visible) | Reviewing draft / processing | None needed |
| writer | `[to orchestrator] All 9 sections done` | Writing complete | Begin assembly |
| editor | `[to orchestrator] All 9 approved` | Editing complete | Confirmed ready to assemble |

### The Shutdown Protocol

At the end of the project, the orchestrator used the formal JSON handshake to terminate agents:

**Orchestrator sends:**
```json
{"type": "shutdown_request", "reason": "Project complete. docs/how-it-works.md assembled. Shut down."}
```

**Agent acknowledges:**
```json
{"type": "shutdown_approved", "requestId": "shutdown-1774548735454@researcher"}
```

**System confirms:**
```
{"type": "teammate_terminated", "message": "researcher has shut down."}
```

### File-Based Handoffs

In addition to messages, agents coordinated through the filesystem:

| From | To | File | Protocol |
|---|---|---|---|
| Researcher | Writer | `docs_research_notes.md` | Researcher writes file, then sends message telling writer the path |
| Writer | Editor | `docs_draft_section_N.md` | Writer writes file, sends message telling editor the filename |
| Writer | Orchestrator | `docs_final_section_N.md` | Writer writes files; orchestrator reads all 9 during assembly |
| Orchestrator | User | `docs/how-it-works.md` | Orchestrator writes final assembled document |

---

## 6. Complete Turn-by-Turn Narrative

### Phase 1: Team Architecture

**Orchestrator Turn 1 — Tool loading**
The orchestrator called `ToolSearch` to load `TeamCreate`, `TaskCreate`, `TaskUpdate`, `TaskList`, and `SendMessage` tools (they were in deferred state and needed loading before use).

**Orchestrator Turn 2 — Pre-work**
Before creating the team, the orchestrator spawned an Explore subagent to read the codebase and understand its structure. This was not a team member — it ran anonymously and returned a comprehensive analysis to inform the agent prompts.

Simultaneously, the orchestrator read `.ignore/agent_team_documentation.md` (the Munger team case study) in full to understand the proven workflow patterns.

**Orchestrator Turn 3 — Team creation**
```
TeamCreate(
  team_name="cco-docs-team",
  description="Agent team to research, write, and edit technical documentation for claude-code-organizer"
)
```
This created:
- Team config at `~/.claude/teams/cco-docs-team/config.json`
- Shared task list namespace at `~/.claude/tasks/cco-docs-team/`

**Orchestrator Turn 4 — Task creation**
Four `TaskCreate` calls, then dependency setup:
```
TaskUpdate(taskId=2, addBlockedBy=["1"])
TaskUpdate(taskId=3, addBlockedBy=["1"])
TaskUpdate(taskId=4, addBlockedBy=["2", "3"])
```

**Orchestrator Turn 5 — Agent spawning**
Three `Agent` tool calls in a single message (all run in background simultaneously):
```
Agent(name="researcher", team_name="cco-docs-team", run_in_background=true, prompt=...)
Agent(name="writer",     team_name="cco-docs-team", run_in_background=true, prompt=...)
Agent(name="editor",     team_name="cco-docs-team", run_in_background=true, prompt=...)
```

All three agents woke up, read the team config, and checked `TaskList`:
- Researcher: found Task #1 unblocked → claimed it immediately
- Writer: found Task #2 blocked by Task #1 → sent idle notification to orchestrator at 17:24:15
- Editor: found Task #3 blocked by Task #1 → began reading research notes proactively anyway

---

### Phase 2: Research

**17:24:15 — Writer first idle notification**
Writer sent `idle_notification` (no summary — just waiting for Task #1). No orchestrator action needed.

**Researcher Turns 1–15 (approximately, during ~17:24 to 17:29):**
The researcher worked invisibly to the orchestrator during this phase — no idle notifications with summaries were sent until the work was done. The researcher:

1. Called `TaskList` → found Task #1 unblocked → claimed it
2. Read `bin/cli.mjs` (1 chunk)
3. Read `src/scanner.mjs` in 5 chunks (~200 lines each)
4. Read `src/mover.mjs` in 2 chunks
5. Read `src/server.mjs` in 3 chunks
6. Read `src/mcp-server.mjs` (1 chunk)
7. Read `src/tokenizer.mjs` (1 chunk)
8. Read `src/ui/index.html` (1 chunk)
9. Read `src/ui/app.js` in 9–10 chunks
10. Read `src/ui/style.css` (partial — for structural information)
11. Read `package.json` (1 chunk)
12. Read `.claude-plugin/plugin.json`, `settings.json`, `skills/organize.md` (3 chunks)
13. Wrote `docs_research_notes.md` — 9-part structured reference document

**17:29:08 — Researcher first meaningful idle notification**
```json
{
  "summary": "[to writer] Research notes complete -- ready at docs_research_notes.md"
}
```

Simultaneously:
- Task #1 was marked `completed` by researcher
- Tasks #2 and #3 automatically unblocked
- Writer received the researcher's message and began reading research notes
- Editor received the Task #1 completion signal and began reading research notes and source files

**17:29:35 — Researcher second idle (quiet)**
No summary — researcher went fully idle after handoff. It would remain idle until the shutdown request.

---

### Phase 3: The Write-Edit Loop

The loop ran 9 times. All 9 sections were approved in exactly 2 rounds.

#### Section 1: Overview & Architecture

| Time | Agent | Action |
|---|---|---|
| 17:29:42 | Writer | Wrote `docs_draft_section_1.md`; SendMessage to editor |
| 17:29:47 | Writer | Idle (awaiting critique) |
| ~17:30 | Editor | Read draft; cross-checked research notes; sent critique |
| 17:31:00 | Writer | Received critique; revised; wrote `docs_final_section_1.md`; SendMessage to editor |
| ~17:31 | Editor | Read final; sent approval |

**Duration (draft → approval):** ~1m 18s
**Editor's critique focus:** Missing pre-flight check description; missing `/cco` auto-install in the CLI section

---

#### Section 2: Scope Discovery Engine

| Time | Agent | Action |
|---|---|---|
| 17:32:13 | Writer | Wrote `docs_draft_section_2.md`; SendMessage to editor |
| ~17:32 | Editor | Read draft; sent critique |
| 17:33:24 | Writer | Revised; wrote `docs_final_section_2.md`; SendMessage to editor |
| ~17:33 | Editor | Read final; sent approval |

**Duration:** ~1m 11s
**Notable:** Section 2 included the full `resolveEncodedProjectPath()` algorithm with code block and step-by-step walkthrough — complex content handled well on first draft.

---

#### Section 3: The 11-Category Scanner

| Time | Agent | Action |
|---|---|---|
| 17:34:45 | Writer | Wrote `docs_draft_section_3.md`; SendMessage to editor |
| ~17:35 | Editor | Read draft; sent critique |
| 17:36:21 | Writer | Revised; wrote `docs_final_section_3.md`; SendMessage to editor |
| ~17:36 | Editor | Read final; sent approval |

**Duration:** ~1m 36s
**Notable:** Most content-dense section — 11 categories each needing filesystem location, parse logic, and special cases. The editor caught the incorrect category name ("settings" instead of "configs") here.

---

#### Section 4: The Mover System

| Time | Agent | Action |
|---|---|---|
| 17:37:30 | Writer | Wrote `docs_draft_section_4.md`; SendMessage to editor |
| ~17:38 | Editor | Read draft; sent critique |
| 17:38:32 | Writer | Revised; wrote `docs_final_section_4.md`; SendMessage to editor |
| ~17:38 | Editor | Read final; sent approval |

**Duration:** ~1m 2s (fastest section)
**Editor's critique focus:** `safeRename()` code snippet had the `isDir` parameter described incorrectly; "atomically" was corrected to describe the actual sequential write-then-delete behavior.

---

#### Section 5: Context Budget Calculator

| Time | Agent | Action |
|---|---|---|
| 17:40:27 | Writer | Wrote `docs_draft_section_5.md`; SendMessage to editor |
| ~17:41 | Editor | Read draft; sent critique |
| 17:42:27 | Writer | Revised; wrote `docs_final_section_5.md`; SendMessage to editor |
| ~17:42 | Editor | Read final; sent approval |

**Duration:** ~2m
**Editor's critique focus:** JSON response shape in the API example had wrong field names and data types (e.g., `items` arrays vs numeric totals in wrong positions).

---

#### Section 6: The Dashboard UI

| Time | Agent | Action |
|---|---|---|
| 17:43:55 | Writer | Wrote `docs_draft_section_6.md`; SendMessage to editor |
| ~17:44 | Editor | Read draft; sent critique |
| ~17:47 | Writer | Revised; wrote `docs_final_section_6.md`; SendMessage to editor |
| ~17:49 | Editor | Read final; sent approval |

**Duration:** ~5m (longest section cycle)
**Notable:** The dashboard UI section is the most complex — REST API table, 3-panel layout, drag-and-drop, undo system, bulk operations, theme system. The ~5 minute cycle reflects the depth of revision required. The Section 7 draft was submitted at 17:49:12, immediately after Section 6 was approved.

---

#### Section 7: MCP Server Mode

| Time | Agent | Action |
|---|---|---|
| 17:49:12 | Writer | Wrote `docs_draft_section_7.md`; SendMessage to editor |
| 17:49:26 | Writer | Idle (14s later — just confirming message sent) |
| ~17:50 | Editor | Read draft; sent critique |
| 17:50:46 | Writer | Revised; wrote `docs_final_section_7.md`; SendMessage to editor |
| ~17:51 | Editor | Read final; sent approval |

**Duration:** ~1m 34s
**Editor's critique focus:** The hardcoded version `"0.5.0"` in the MCP server was omitted; added as a notable implementation detail.

---

#### Section 8: Plugin & Skill System

| Time | Agent | Action |
|---|---|---|
| 17:57:05 | Writer | Wrote `docs_draft_section_8.md`; SendMessage to editor |
| ~17:58 | Editor | Read draft; sent critique |
| 18:02:58 | Writer | Revised; wrote `docs_final_section_8.md`; SendMessage to editor |
| ~18:03 | Editor | Read final; sent approval |

**Duration:** ~5m 53s (largest revision gap)
**Notable:** The `~5m` revision time is the longest in the project. The editor likely requested significant additions — the skill description mismatch and fabricated `--experimental-vm-modules` flag both lived here. Correcting these required re-reading the source files, not just editing prose.

---

#### Section 9: Technologies & Dependencies

| Time | Agent | Action |
|---|---|---|
| 18:05:55 | Writer | Wrote `docs_draft_section_9.md`; SendMessage to editor |
| ~18:06 | Editor | Read draft; sent critique |
| 18:06:31 | Writer | Revised; wrote `docs_final_section_9.md`; SendMessage to editor |
| 18:06:46 | Writer | Idle |
| 18:06:48 | Editor | `[to writer] Section 9 approved - all 9 sections complete` |

**Duration:** ~36s (fastest revision in the project)
**Notable:** The quality arc completed here — the writer's fastest revision, and the editor's cleanest approval. By Section 9, the writer had internalized the editor's standards and was pre-applying them before submission.

**After Section 9 approval:**
- Writer called `TaskUpdate(taskId=2, status=completed)` → Task #2 complete
- Editor called `TaskUpdate(taskId=3, status=completed)` → Task #3 complete
- Task #4 automatically unblocked
- Both writer and editor sent notifications to the orchestrator

---

### Phase 4: Assembly

**Orchestrator received `idle_notification` from writer** with summary: "[to orchestrator] All 9 sections written and approved"
**Orchestrator received notification from editor**: "All 9 sections reviewed and approved."

**Orchestrator checked `TaskList`:**
```
#1 [completed]
#2 [completed]
#3 [completed]
#4 [pending] — now unblocked
```

**Orchestrator claimed and executed Task #4:**
1. `TaskUpdate(taskId=4, owner="team-lead", status="in_progress")`
2. Read all 9 final section files in parallel (3 at a time)
3. Assembled into `docs/how-it-works.md` with:
   - H1 title and intro paragraph
   - Table of contents with anchor links
   - All 9 sections as H2 sections with H3 subsections
   - Consistent formatting throughout
   - Footer crediting the team
4. `TaskUpdate(taskId=4, status="completed")`

---

### Phase 5: Shutdown

**Orchestrator → All agents:**
```
SendMessage(to="researcher", message={"type": "shutdown_request", "reason": "Project complete..."})
SendMessage(to="writer",     message={"type": "shutdown_request", "reason": "Project complete..."})
SendMessage(to="editor",     message={"type": "shutdown_request", "reason": "Project complete..."})
```

**Writer** (18:12:20): Confirmed `shutdown_approved`. System: `teammate_terminated: writer`.

**Editor** (18:12:20): Multiple idle notifications between 18:07 and 18:12 (see Edge Case 1 below), then confirmed `shutdown_approved`. System: `teammate_terminated: editor`.

**Researcher** (18:12:26): Confirmed `shutdown_approved`. System: `teammate_terminated: researcher`.

---

## 7. All Artifacts Produced

### Intermediate Files (Working Documents)

| File | Size (est.) | Producer | Purpose |
|---|---|---|---|
| `.ignore/docs_research_notes.md` | ~40 KB | Researcher | Source of truth for all writing |
| `.ignore/docs_draft_section_1.md` | — | Writer | First draft — overview |
| `.ignore/docs_draft_section_2.md` | — | Writer | First draft — scope discovery |
| `.ignore/docs_draft_section_3.md` | — | Writer | First draft — scanner |
| `.ignore/docs_draft_section_4.md` | — | Writer | First draft — mover |
| `.ignore/docs_draft_section_5.md` | — | Writer | First draft — context budget |
| `.ignore/docs_draft_section_6.md` | — | Writer | First draft — dashboard UI |
| `.ignore/docs_draft_section_7.md` | — | Writer | First draft — MCP server |
| `.ignore/docs_draft_section_8.md` | — | Writer | First draft — plugin/skill |
| `.ignore/docs_draft_section_9.md` | — | Writer | First draft — technologies |
| `.ignore/docs_final_section_1.md` | ~1.5 KB | Writer (post-critique) | Approved section 1 |
| `.ignore/docs_final_section_2.md` | ~2.5 KB | Writer (post-critique) | Approved section 2 |
| `.ignore/docs_final_section_3.md` | ~3.5 KB | Writer (post-critique) | Approved section 3 |
| `.ignore/docs_final_section_4.md` | ~3.5 KB | Writer (post-critique) | Approved section 4 |
| `.ignore/docs_final_section_5.md` | ~3.0 KB | Writer (post-critique) | Approved section 5 |
| `.ignore/docs_final_section_6.md` | ~4.0 KB | Writer (post-critique) | Approved section 6 |
| `.ignore/docs_final_section_7.md` | ~2.5 KB | Writer (post-critique) | Approved section 7 |
| `.ignore/docs_final_section_8.md` | ~2.5 KB | Writer (post-critique) | Approved section 8 |
| `.ignore/docs_final_section_9.md` | ~2.5 KB | Writer (post-critique) | Approved section 9 |

### Final Deliverable

| File | Size (est.) | Pages | Producer |
|---|---|---|---|
| `docs/how-it-works.md` | ~30 KB / ~4,500 words | 9 sections | Orchestrator (assembled from finals) |

---

## 8. Inter-Agent Message Log

A complete record of every significant `SendMessage` call during the project:

| # | From | To | Summary | Trigger |
|---|---|---|---|---|
| 1 | Researcher | Writer | "Research notes complete at docs_research_notes.md" | Task #1 done |
| 2 | Writer | Editor | "Section 1 draft ready — please read docs_draft_section_1.md" | Draft written |
| 3 | Editor | Writer | Section 1 critique | Draft read |
| 4 | Writer | Editor | "Revised Section 1 ready" | Revision written |
| 5 | Editor | Writer | "Section 1 approved — proceed to Section 2" | Revision approved |
| 6–23 | Writer/Editor | Editor/Writer | Sections 2–9 draft / critique / revise / approve (×4 per section) | Loop ×8 |
| 24 | Writer | Orchestrator | "All 9 sections written and approved" | Section 9 done |
| 25 | Editor | Orchestrator | "All 9 sections reviewed and approved" | Section 9 approved |
| 26 | Orchestrator | Researcher | `shutdown_request` | Assembly complete |
| 27 | Orchestrator | Writer | `shutdown_request` | Assembly complete |
| 28 | Orchestrator | Editor | `shutdown_request` | Assembly complete |
| 29 | Writer | Orchestrator | `shutdown_approved` | Received shutdown |
| 30 | Editor | Orchestrator | `shutdown_approved` | Received shutdown |
| 31 | Researcher | Orchestrator | `shutdown_approved` | Received shutdown |

**Total SendMessage calls: ~31**
**Peer-to-peer messages (writer ↔ editor): ~24 (about 2.7 per section × 9 sections)**
**Orchestrator → Agent messages: 3 (shutdown only)**

---

## 9. Quality Metrics — The Write-Edit Loop

### Time Per Section

| Section | Draft submitted | Revision submitted | Cycle time | Notes |
|---|---|---|---|---|
| 1 | 17:29:42 | 17:31:00 | ~1m 18s | Baseline |
| 2 | 17:32:13 | 17:33:24 | ~1m 11s | Complex algorithm, handled cleanly |
| 3 | 17:34:45 | 17:36:21 | ~1m 36s | Longest section (11 categories) |
| 4 | 17:37:30 | 17:38:32 | ~1m 02s | Fastest — mover system well-documented |
| 5 | 17:40:27 | 17:42:27 | ~2m 00s | JSON shape corrections took time |
| 6 | 17:43:55 | ~17:47 | ~5m 00s | Most complex section (UI) |
| 7 | 17:49:12 | 17:50:46 | ~1m 34s | MCP section concise |
| 8 | 17:57:05 | 18:02:58 | ~5m 53s | Longest — fabricated detail caught and fixed |
| 9 | 18:05:55 | 18:06:31 | ~36s | Fastest revision — writer pre-applying standards |

### Quality Arc

All 9 sections were approved in exactly 2 rounds (draft → critique → revision → approval). No section required a third round.

The progression from Section 1 (1m 18s) through Section 9 (36s) shows the writer internalizing the editor's standards over the course of the project. By Section 9, the revision was nearly instant — the editor had very little to correct.

---

## 10. Key Design Principles Illustrated

### 10.1 Agents Communicate Only Through Messages and Files — Never Directly

No agent calls another agent's functions or reads another agent's internal state. The writer cannot compel the editor to act — it can only write a file and send a message. The editor cannot rewrite the draft — it can only send critique and let the writer decide what to change. This **loose coupling** makes each agent independently testable, replaceable, and understandable.

### 10.2 The Task List Is Shared Ground Truth

Every agent had access to the same `TaskList`. Dependencies (`blockedBy`) were structural constraints, not conventions. This meant:
- The orchestrator did not need to tell writer and editor "wait for the researcher" — the task system enforced it
- Any agent could check overall project state at any time
- Task completions automatically unlocked downstream work

The orchestrator made **zero intervention messages** during the write-edit loop. The task system handled all coordination.

### 10.3 `idle_notification` Is the Heartbeat — Not a Problem Signal

New practitioners often misread idle notifications as errors. In this system, idle was the **normal resting state** between turns. Every agent went idle after each processing burst — after writing a file, after sending a message. The summary field on peer-message idles gave the orchestrator passive visibility into all 18 write-edit sub-turns without requiring active monitoring.

### 10.4 Pre-Work Before Spawning Produces Better Agent Prompts

The orchestrator ran an Explore subagent and read the team workflow template *before* creating the team. This meant the agent prompts were specific, accurate, and actionable — they named actual file paths, actual function names, and accurate file sizes. Agents that receive well-informed prompts make fewer errors and require less correction.

### 10.5 Specialisation Matters — Capability Must Match Task

The researcher was correctly specialised for reading and synthesising source code. The writer was correctly specialised for producing clear technical prose. The editor was correctly specialised for accuracy verification. None of them tried to do each other's jobs. The orchestrator was correctly positioned as a coordinator — it assembled the document itself when that was the most efficient path, rather than spawning a specialist.

### 10.6 The Feedback Loop Improves Quality Over Time

18 editorial cycles across 9 sections produced measurable quality improvement — from 1m 18s revision cycles to 36s. This is emergent: neither the writer nor the editor was told "your quality is improving." The improvement came from the writer receiving specific, concrete feedback and applying it to the next section before submitting.

### 10.7 Parallel Spawning Is More Efficient Than Sequential

All 3 agents were spawned in a single message. This meant all three were alive and ready from the moment the team was created. The writer and editor were correctly idle (blocked by Task #1) rather than not-yet-existing. When Task #1 completed, they were **immediately available** — no spawn latency at that point.

### 10.8 File Paths Are First-Class Communication

Half the coordination in this system was file-path-based, not message-based. The researcher told the writer: "The research notes are at `.ignore/docs_research_notes.md`." The writer told the editor: "Please read `docs_draft_section_3.md`." The naming convention `docs_draft_section_N.md` / `docs_final_section_N.md` was essential — the editor always knew exactly which file to read.

---

## 11. What Could Go Wrong — Edge Cases Encountered

### Edge Case 1: Editor Sent Multiple Idle Notifications After Shutdown Request

**Problem:** After the orchestrator sent shutdown requests, the editor sent ~20 idle notifications at 3-second intervals between 18:07:03 and 18:07:55 before finally confirming `shutdown_approved` at 18:12:20.

The likely cause: the editor was in the middle of a post-approval processing turn when the shutdown request arrived. It entered a loop, possibly trying to check for new tasks or verify the full project state before accepting shutdown.

**Resolution:** The orchestrator received the many idle notifications, did not react to them (correctly), and waited for the `shutdown_approved` confirmation. The editor eventually processed the shutdown request and terminated cleanly.

**Lesson:** Shutdown requests may not be processed immediately if an agent is mid-turn. Always wait for the `teammate_terminated` system confirmation rather than assuming shutdown happened after sending the request.

### Edge Case 2: Editor Sent Confusing Post-Approval Message

**Problem:** At 18:07:03, after all 9 sections were already approved, the editor sent a message to the writer: "[to writer] Section 1 was already approved earlier." This was presumably the editor re-processing its own state and sending a spurious clarification.

**Resolution:** The writer was already idle awaiting shutdown. The orchestrator had already received the all-approved notification. The message was benign noise.

**Lesson:** In iterative loops, agents may occasionally send redundant messages as they reconcile state. Design orchestrators to be idempotent — receiving the same approval notification twice should not trigger duplicate assembly.

---

## 12. Summary Statistics

### Team Composition

| Agent | Type | Tasks | Status |
|---|---|---|---|
| Orchestrator | Root session | Architecture + monitoring + assembly | Active throughout |
| Researcher | General-purpose | Task #1 | ✅ Shut down |
| Writer | General-purpose | Task #2 | ✅ Shut down |
| Editor | General-purpose | Task #3 | ✅ Shut down |

### Task Metrics

| Metric | Value |
|---|---|
| Tasks created | 4 |
| Task dependency edges | 4 |
| Task completions | 4 |
| Orchestrator interventions during write-edit loop | 0 |
| Agent self-reported completions | 3 (researcher + writer + editor) |
| Orchestrator task completions | 1 (Task #4 assembly) |

### Communication Metrics

| Metric | Value |
|---|---|
| Total `SendMessage` calls | ~31 |
| Peer-to-peer messages (writer ↔ editor) | ~24 |
| Orchestrator → Agent messages | 3 (shutdown only) |
| Agent → Orchestrator messages | 2 (all-done notifications) |
| `idle_notification` events received | ~40+ |
| Idle events requiring orchestrator action | 0 |
| Shutdown handshakes | 3 |

### Content Metrics

| Metric | Value |
|---|---|
| Source files read by researcher | 12 |
| Source lines read | ~6,000+ |
| Research notes produced | ~40 KB |
| Draft sections written | 9 |
| Final sections written | 9 |
| Editorial cycles | 18 (2 per section) |
| Sections requiring 3+ rounds | 0 |
| Inaccuracies caught by editor | 7 categories |

### Final Deliverable

| Metric | Value |
|---|---|
| Document sections | 9 |
| Approximate word count | ~4,500 words |
| Code blocks | 15+ |
| Tables | 12+ |
| Total elapsed time | ~45 minutes |
| Orchestrator interventions | 0 (zero, end-to-end) |

---

*Documentation produced by Claude (Sonnet 4.6) — March 2026*
*Project: Claude Code Organizer Technical Documentation — `cco-docs-team`*
