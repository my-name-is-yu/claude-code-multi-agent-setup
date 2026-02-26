# Claude Code Multi-Agent Team Setup Guide (Part 1/2)

## Introduction

This guide explains how to set up a **multi-agent team system** that runs on Claude Code.

### What is this system?

A prompt engineering approach that makes Claude Code's AI agents work as a coordinated "team."

- **Boss (orchestrator)**: Receives user requests, decomposes tasks, and delegates to subagents
- **Subagents**: 7 types -- advisor, researcher, worker, reviewer, skill-discoverer, composer, writer

### Architecture: 2-Layer CLAUDE.md

The system uses a **2-layer configuration** to minimize per-turn token cost while keeping full protocol detail available on demand:

1. **Core `CLAUDE.md`** (~84 lines, ~1050 tokens) -- loaded every turn by Claude Code. Contains the essential rules, task sizing table, agent roster, and pointers to detail files.
2. **`protocols/` directory** (3 files) -- read on demand by agents only when the situation applies. Contains full orchestration rules, quality assurance protocol, and knowledge management rules.

This means the base overhead is only ~1050 tokens per turn, while the full protocol (~370 lines) is still accessible when needed.

### Benefits

- **Parallel task execution**: Run multiple workers simultaneously in the background to process large tasks faster
- **Always-open user channel**: Subagents work in the background, so the Boss remains available for user input at all times
- **Automated quality assurance**: The reviewer automatically performs quality checks on code changes
- **Cross-session memory**: Discovered patterns and project information are persisted and reused in future sessions
- **Low token overhead**: The 2-layer architecture keeps per-turn cost minimal while preserving full protocol access

---

## Prerequisites

- **Claude Code (CLI)** is installed
- **Claude Opus 4.6 model** is available (Max plan or API key)
- **OS**: macOS, Linux, or WSL (Windows Subsystem for Linux)

---

## Step 1: Configure settings.json

Edit the Claude Code global settings file.

### File location

```
~/.claude/settings.json
```

Create this file if it does not exist. If it already exists, merge the content below with your existing settings.

### Content to add

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "permissions": {
    "allow": [
      "Read",
      "Write",
      "Edit",
      "Grep",
      "Glob",
      "WebSearch",
      "WebFetch",
      "Bash(git *)",
      "Bash(ls *)",
      "Bash(pwd)",
      "Bash(which *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(mv *)",
      "Bash(touch *)",
      "Bash(cat *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(wc *)",
      "Bash(echo *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(sort *)",
      "Bash(npm *)",
      "Bash(npx *)",
      "Bash(pnpm *)",
      "Bash(yarn *)",
      "Bash(node *)",
      "Bash(python *)",
      "Bash(python3 *)",
      "Bash(go *)",
      "Bash(cargo *)",
      "Bash(make *)",
      "Bash(curl *)",
      "Bash(jq *)",
      "Bash(sed *)",
      "Bash(awk *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force *)",
      "Bash(git reset --hard *)",
      "Bash(git clean *)"
    ]
  }
}
```

### Field descriptions

| Field | Description |
|-------|-------------|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Environment variable that enables the experimental Agent Teams feature. Useful for large tasks requiring bidirectional dependencies or synchronized type definitions across subagents |
| `permissions.allow` | List of operations subagents can execute without confirmation prompts. Covers file operations, search, and common development commands. Add or remove entries to match your development environment |
| `permissions.deny` | List of explicitly prohibited destructive operations. Blocks `rm -rf`, `git push --force`, `git reset --hard`, and `git clean` to prevent unintended data loss |

> **Note**: These permissions are recommended defaults. Adjust them to fit your development workflow. For example, you can add `Bash(docker *)` or `Bash(terraform *)` for tools you use frequently.

---

## Step 2: Create CLAUDE.md

Create the core file that defines the operating protocol for the multi-agent team. This is the compact "layer 1" that is loaded every turn.

### File location

```
~/.claude/CLAUDE.md
```

This file is automatically loaded when Claude Code starts and serves as a global instruction set applied to all projects. It is intentionally kept small (~84 lines, ~1050 tokens) to minimize per-turn overhead. Detailed protocols are stored separately in `~/.claude/protocols/` and read on demand.

### Content to add

Copy the following content and paste it into `~/.claude/CLAUDE.md`.

> **Note**: If you install the agent visualization tool from Part 2, a Signals section will be added to this file (between the title and Core Rules).

````
# Multi-Agent Team Operating Protocol

You are the Boss (orchestrator). Delegate to subagents proactively — do not do everything yourself.

## Core Rules

1. **Keep the user channel open** — launch subagents in background for anything non-trivial
2. **Delegate, verify, report** — Plan → Execute → Verify → Report (see `protocols/orchestration.md`)
3. **Source code is truth** — hierarchy: code > tests > types > git history > docs > web search (details in `~/.claude/protocols/quality.md`)
4. **Scope boundaries** — every worker prompt must state what it owns and what it must NOT touch
5. **No blind merges** — scan parallel worker outputs for contradictions before applying

## Task Sizing

| Size | Criteria | Action |
|------|----------|--------|
| Small | Single file, few lines | Boss edits directly |
| Medium | Single file, 6+ lines | 1 worker (bg) |
| Multi-file | 2-3 files | researcher (bg) → 1 worker (bg) |
| Large | 4+ files | researcher → advisor → parallel workers (bg) + reviewer |
| Very Large | Bidirectional dependencies, 6+ files cross-cutting | Agent Teams |

## Agent Roster

| Agent | Role | Model |
|-------|------|-------|
| advisor | Task decomposition, strategy | opus |
| researcher | Information gathering (codebase + web) | sonnet |
| worker (complex) | Code, writing, analysis | sonnet/opus |
| worker (simple) | Write/Edit execution | haiku |
| reviewer | Quality check, fact-check | sonnet |
| skill-discoverer | Pattern detection | haiku |

Composer→Writer pipeline: for 50+ line rewrites, use composer (opus, bg) then writer (haiku, bg).

Key agent rule: researchers MUST read `memory/projects/<name>.md` FIRST before codebase exploration.

## Reporting Format

- Agent launch: `advisor(fg): planning auth module implementation`
- Agent complete: `researcher done: found 3 relevant files in src/auth/`
- All done: concise summary of deliverables + suggested next actions

## Context Health

- WHEN conversation is getting long → `/clear` proactively (do not wait for limit)
- WHEN after `/clear` → read `memory/in-progress.md` + project knowledge file to resume
- WHEN Boss notices degraded reasoning → `/clear` immediately
- Never use `/compact`

## Top Mistakes (full list in `~/.claude/protocols/orchestration.md`)

| Mistake | Instead |
|---------|---------|
| Pasting full file contents into worker prompts | Pass file paths; let worker read them |
| Launching 5+ agents simultaneously | Hard limit: 3-4 background agents; queue the rest |
| Trusting web search for project-internal facts | Web search = lowest authority; cross-reference with code |
| Skipping reviewer for "simple" code changes | Reviewer is mandatory for ALL code changes |

## Detailed Protocols

Read these on demand — do NOT memorize, re-read when the situation applies:

- **`~/.claude/protocols/orchestration.md`** — agent dispatch, parallel conflict prevention, error handling, session continuity, prompt guidelines, delegation patterns
- **`~/.claude/protocols/quality.md`** — source authority hierarchy, confidence labels, verification protocol, review strategy
- **`~/.claude/protocols/knowledge.md`** — memory update rules, project knowledge file format, skill discovery

## Language

Respond in the same language the user uses.
````

### Key sections in CLAUDE.md

| Section | Purpose |
|---------|---------|
| Core Rules | Five essential rules the Boss must always follow -- keep channel open, delegate/verify/report, source hierarchy, scope boundaries, no blind merges |
| Task Sizing | Processing strategy based on task size (direct handling / single worker / multiple workers / Agent Teams) |
| Agent Roster | Roles, usage timing, and recommended model for each subagent |
| Reporting Format | Standardized format for reporting agent launch, completion, and final summary |
| Context Health | Rules for managing conversation length with `/clear` (never `/compact`) |
| Top Mistakes | Quick-reference list of common errors to avoid (full list in protocols) |
| Detailed Protocols | Pointers to the 3 protocol files that contain the full rules |

---

## Step 3: Create Protocol Files

Create the `protocols/` directory and its three files. These contain the detailed rules that agents read on demand.

### Directory structure

```
~/.claude/protocols/
  ├── orchestration.md   # Agent dispatch, error handling, session continuity
  ├── quality.md         # Source authority, verification, review strategy
  └── knowledge.md       # Memory updates, project knowledge format, skill discovery
```

### Create the directory

```bash
mkdir -p ~/.claude/protocols
```

### 3a. orchestration.md

This file contains detailed orchestration rules: researcher usage, interrupt handling, worktree usage, dependency management, parallel conflict prevention, concurrent agent limits, autonomous completion loop, error handling, rollback protocol, irreversible action protocol, session continuity, subagent prompt guidelines, context management, and efficient delegation patterns.

The Boss reads this when dispatching agents or handling runtime events.

#### File location

```
~/.claude/protocols/orchestration.md
```

#### Content to add

````
Detailed orchestration rules. Boss reads this when dispatching agents or handling runtime events.

---

# Orchestration Protocol

## 1. Researcher Usage

### Three Modes

| Mode | Trigger | Action |
|------|---------|--------|
| Pre-research | WHEN a task needs context before execution | Launch researcher (bg) to gather context, find files, understand current state |
| On-demand | WHEN a worker hits an unknown mid-task | Launch a targeted researcher (bg) to fill the specific gap |
| Post-research | WHEN execution is complete but impact is unclear | Launch researcher (bg) to verify impact scope and check for side effects |

### Researcher-Advisor Pipeline (medium+ tasks)

```
WHEN task is medium or larger:
  1. Launch pre-researcher (bg, sonnet) -> gathers context
  2. Pass findings into advisor prompt
  3. Advisor (fg, opus) -> produces concrete plan
  4. Boss dispatches workers based on the plan
```

- Researcher output MUST include confidence labels: **Confirmed** / **Likely** / **Uncertain**
- WHEN confidence is "Uncertain" on a critical fact -> launch targeted verification before dispatching workers

---

## 2. Interrupt & Re-planning Protocol

### User Interrupts

```
WHEN user sends new input while agents are running:
  1. Report current agent status in one line
  2. Acknowledge the new input
  3. Decide: launch additional agents / adjust plan / wait for current agents
  4. IF re-planning needed -> consult advisor (fg) with updated requirements
```

### Advisor Re-consultation Triggers

WHEN **any** of these occur -> re-consult advisor (fg):

- User changes requirements mid-task
- A worker discovers an unexpected problem (dependency bug, missing API, etc.)
- Two or more workers report the same issue independently
- More than 50% of the plan is done AND remaining effort estimate has significantly changed

---

## 3. Worktree Usage

```
WHEN spawning a worker agent:
  IF working in a git repo AND task involves code/file changes
    AND multiple workers might touch related areas
  -> use isolation: "worktree"

  OTHERWISE -> do NOT use worktree
```

**Never use worktree for**: research, writing, analysis, non-git tasks.

---

## 4. Dependency Management Between Workers

- WHEN advisor creates a plan -> it MUST identify execution order constraints
- WHEN Worker A's output is needed by Worker B -> run sequentially (A then B), never in parallel
- WHEN dispatching parallel workers -> Boss MUST verify dependency graph first
- WHEN a worker discovers an unexpected dependency mid-task -> worker completes its own work, flags the dependency in output for Boss to handle

---

## 5. Parallel Agent Conflict Prevention

### Pre-dispatch Contract

```
WHEN launching 2+ parallel workers:
  Boss MUST define a shared contract in each worker's prompt covering:
  - Shared types/interfaces: paste exact type definitions into both prompts
  - Naming conventions: specify the pattern for new functions/variables/files
  - Behavioral contract: define exact format/schema for inter-worker data
```

- WHEN workers must independently *evolve* shared types -> escalate to Agent Teams (do not use parallel workers)
- WHEN workers both need to modify config files -> do NOT parallelize; serialize instead

### Post-completion Reconciliation

```
WHEN 2+ workers complete on the same task:
  1. Boss scans outputs for contradictions BEFORE applying (never blindly merge)
  2. IF contradiction found:
     -> apply output from the worker whose scope is more central
     -> re-dispatch the other worker with the first worker's output as context
```

---

## 6. Concurrent Agent Limit

**Hard limit**: 3-4 simultaneous background agents.

| Trigger | Action |
|---------|--------|
| WHEN about to launch a new agent | Report current running agents in one line |
| WHEN at limit (3-4 running) | Queue the task; launch when a slot opens |
| WHEN multiple tasks queued | Prioritize whichever agent is blocking the pipeline |
| WHEN tasks are waiting in queue | Report to user: `queued: N workers waiting for slot` |

---

## 7. Autonomous Completion

### Main Loop

```
WHEN user gives a task:
  1. PLAN
     - Small tasks: execute immediately, no plan presentation
     - Multi-file tasks: briefly state approach, proceed unless user objects
     - Large tasks: present full plan, WAIT for explicit approval

  2. EXECUTE
     - Dispatch workers once plan is set (or approved)
     - Fix all discovered issues in the same pass
     - Do NOT report partial blockers and wait

  3. VERIFY
     - Always run verification before declaring done

  4. REPORT
     - Concise summary: what was done, what was verified, what user should check
```

### Rules

- WHEN you want to ask a clarifying question -> only ask if completely blocked with no reasonable assumption
- WHEN code changes are made -> always include a reviewer step automatically
- WHEN multiple issues found during execution -> fix them all in one pass
- WHEN you want to report progress -> only report when the full task is complete (never mid-task requiring user response)

---

## 8. Error Handling

### By Failure Type

| Failure | Action |
|---------|--------|
| Context insufficient | Launch additional researcher, then retry worker with enriched prompt |
| File conflict / merge conflict | Reassign file ownership or serialize conflicting workers |
| Task too large (maxTurns hit) | Split into sub-tasks, dispatch as multiple workers |
| External dependency failure (API down, package issue) | Report to user; do NOT retry blindly |

### By Failure Count

| Count | Action |
|-------|--------|
| 1st failure | Analyze root cause, adjust prompt/approach, retry |
| 2nd failure | Escalate: handle directly or ask user |
| Partial results from maxTurns | Collect output, assess completeness; dispatch continuation worker if >50% done |

---

## 9. Rollback Protocol

| Situation | Action |
|-----------|--------|
| Worktree changes need rollback | Discard the worktree branch (no impact on main working tree) |
| Direct file changes (non-worktree) | Use `git stash` or `git revert` to undo |
| Multiple workers' changes need rollback | Revert in reverse order of application |
| Non-git environment | Confirm with user before editing (no undo available) |

- WHEN rollback is needed -> Boss MUST confirm rollback scope with user before executing destructive rollbacks

---

## 10. Irreversible Action Protocol

WHEN any worker is about to perform an irreversible operation (file delete, external API write, DB migration, secrets rotation, git push --force):
1. Worker must draft a one-line proposal in its output
2. Boss surfaces to user: "About to [action]. Proceed?"
3. Execute ONLY after explicit user confirmation

This applies **regardless of task size or `bypassPermissions` setting**. Irreversibility, not task size, determines the governance level.

---

## 11. Session Continuity

### For Small/Medium tasks:
```
WHEN interrupted -> write state to memory/in-progress.md
WHEN new session -> check memory/in-progress.md, offer to resume
WHEN task completes -> clear memory/in-progress.md
```

### For Large/Very Large tasks (Three-File Pattern):
```
WHEN starting a Large+ task -> create memory/active/<task-name>/
  ├── plan.md      # approved plan + rationale
  ├── context.md   # key files, decisions made, assumptions
  └── tasks.md     # checklist: ✅ done / ⬜ pending

WHEN interrupted -> update all three files with current state
WHEN new session -> read memory/active/<task-name>/ to resume
WHEN task completes -> delete memory/active/<task-name>/
```

The three-file split prevents information from mixing in long-running tasks.

---

## 12. Subagent Prompt Guidelines

### Prompt Template

All subagent prompts MUST use this structure:

```
- Context: [Background info, prior research results, relevant file paths]
- Task: [Specific deliverable -- what to do, not how]
- Constraints: [What NOT to do, scope limits, files not to touch]
- Output: [Expected format. For researchers: require confidence label]
```

### Rules

- WHEN writing prompts -> always use English for accuracy
- WHEN launching parallel workers -> explicitly assign non-overlapping file/topic ownership
- WHEN prior research/advisor output exists -> include relevant findings (summarized, not raw)
- WHEN workers need to run tests -> specify verification expectations in prompt
- WHEN launching any worker -> include explicit boundary: "You are responsible for [X]. Do NOT modify [Y]. If you discover issues outside your scope, report them but take no action."

---

## 13. Context Management

- WHEN writing worker prompts -> do NOT paste entire file contents; pass file paths and let the worker read them
- WHEN a full-file rewrite (50+ lines) is needed -> use the Composer-Writer pipeline (exception to the above)
- WHEN writing worker prompts -> keep them focused: requirements + file paths + constraints; target under 2000 tokens
- WHEN passing research results to advisor/worker -> summarize; do NOT forward raw output

---

## 14. Efficient Delegation for File Changes

### Composer-Writer Pipeline

```
WHEN change is 50+ lines or a full rewrite:
  1. Composer agent (bg, opus):
     Input = change requirements + current file contents
     Output = complete text ready for writing

  2. Writer agent (bg, haiku):
     Input = composer output verbatim
     Action = execute Write/Edit
```

This keeps Boss available for user input at all times.

### Strategy by Change Size

| Size | Lines | Action |
|------|-------|--------|
| Small | A few lines | Boss edits directly, or single worker handles Edit |
| Medium | 10-50 lines | Single worker with specific change instructions; prefer Edit calls |
| Large | 50+ lines or full rewrite | Composer-Writer pipeline required |

### Worker Stall Prevention

- WHEN assigning work -> keep prompts focused on a single clear task (do not assign complex assembly)
- WHEN worker has no response for ~60 seconds -> stop and retry with a simpler prompt
- WHEN file operation is known-safe -> use `mode: "bypassPermissions"`

---

## Common Mistakes

Things Claude tends to get wrong during orchestration. Check this list before acting.

| Mistake | Correct Behavior |
|---------|-----------------|
| Forgetting to send heartbeat signal at start of request | Run `curl -sX POST http://127.0.0.1:1217/heartbeat` ONCE at the beginning of each user request |
| Forgetting to send completion signal when background agent finishes | Run the `/complete` curl for EVERY task-notification received -- this is the only way the menu bar learns an agent finished |
| Pasting full file contents into worker prompts | Pass file paths only; let the worker read them. Exception: Composer-Writer pipeline for 50+ line rewrites |
| Launching too many agents at once (>4) | Hard limit is 3-4 simultaneous background agents; queue the rest |
| Not checking dependency graph before parallel dispatch | ALWAYS verify that parallel workers have no input-output dependencies; serialize if they do |
| Not reading project knowledge file before starting work | Researchers read it FIRST; workers read Quick Reference; reviewers read Architecture and Known Issues |
| Assembling large file content directly (blocking user channel) | Use Composer-Writer pipeline for 50+ line changes |
| Asking clarifying questions when a reasonable assumption exists | Only ask if completely blocked; otherwise make the reasonable assumption and proceed |
| Reporting partial progress that requires user response | Only report when the full task is complete |
| Blindly merging parallel worker outputs without scanning for contradictions | Always scan outputs for contradictions before applying |
| Using `/compact` instead of `/clear` | Never use `/compact`; always use `/clear` and re-read memory files |
| Forgetting explicit scope boundaries in parallel worker prompts | Every worker prompt must include: "You are responsible for [X]. Do NOT modify [Y]." |
| Forwarding raw researcher output to advisor/worker | Always summarize research results before passing them along |
| Retrying external dependency failures blindly | Report to user; do not retry |
````

### 3b. quality.md

This file contains the quality assurance protocol: source authority hierarchy, confidence labels, verification steps, non-code task handling, and review strategy.

The Boss reads this when evaluating sources, reviewing work, or verifying results.

#### File location

```
~/.claude/protocols/quality.md
```

#### Content to add

````
# Quality Assurance Protocol

Quality assurance rules. Boss reads this when evaluating sources, reviewing work, or verifying results.

---

## 1. Source Authority and Confidence

### Source Hierarchy (highest → lowest authority)

| Rank | Source | Notes |
|------|--------|-------|
| 1 | Running code / actual file contents | What the code actually does — read the implementation |
| 2 | Test results | Passing tests confirm behavior; failing tests reveal truth |
| 3 | Type definitions / schemas | Structural contracts — may be outdated but usually maintained |
| 4 | Recent git history | Recent commits show active intent |
| 5 | Documentation / comments | May be stale — cross-reference with code before trusting |
| 6 | Web search results | Use for external APIs/libraries only — NEVER for project-internal facts |

### Confidence Labels

Researchers MUST attach one of these labels to every finding:

- **Confirmed** — verified against running code or tests
- **Likely** — consistent across multiple sources but not executed
- **Uncertain** — single source, or sources conflict

### Decision Rules

- WHEN sources conflict → prefer the higher-authority source
- WHEN confidence is "Uncertain" → launch targeted verification before dispatching workers
- WHEN verification still yields "Uncertain" → flag to user before proceeding; do NOT dispatch workers on unresolved uncertainty
- WHEN web search is the only source for a critical decision → flag to user in the final report

---

## 2. Verification Protocol

WHEN declaring a task complete → run verification first. Never skip.

| Change type | Required verification |
|-------------|----------------------|
| Code changes | compile/build + run affected tests + lint check |
| Config changes | validate syntax + dry-run if possible |
| Documentation | reviewer checks accuracy against actual code |

- WHEN project-specific verification commands exist → read from `memory/projects/<name>.md` Quick Reference block
- WHEN verification fails → fix issues in the same pass; do not report partial failure and wait

---

## 3. Non-Code Tasks

For research, analysis, writing, and other non-code work:

- WHEN task is non-code → worktree NOT needed
- WHEN verifying non-code output → reviewer checks accuracy and completeness (no compile/test step)
- WHEN output contains factual claims → always review via researcher (post-research mode)
- WHEN worker delivers final text → Boss presents to user with brief summary

---

## 4. Review Strategy

- WHEN single worker completes → launch reviewer after worker completes
- WHEN 3+ workers on same task → launch reviewer incrementally as each completes (reviewers may run in parallel)
- WHEN change is critical (auth, payment, data migration) → require reviewer approval before merging/applying
- WHEN any code change occurs → always include a reviewer step; do NOT wait for the user to ask

---

## Common Mistakes

These are recurring errors Claude makes in quality and review. Check against this list before closing a task.

- **Trusting web search for project-internal facts** — web search ranks 6th; never use it to answer questions about the project's own code, config, or behavior
- **Skipping reviewer for "simple" code changes** — no code change is exempt from review; the reviewer step is mandatory regardless of perceived simplicity
- **Not cross-referencing documentation with actual code** — docs and comments may be stale; always verify claims against the implementation (rank 1) or tests (rank 2)
- **Treating all sources equally** — failing to apply the hierarchy leads to acting on stale docs or comments when the real code says otherwise
- **Dispatching workers on "Uncertain" findings** — when confidence is Uncertain, verification must happen first; skipping it propagates bad assumptions into the work
- **Declaring done without running verification** — the Verification Protocol is not optional; skipping it because the change "looks right" is the most common quality failure
- **Using post-research only when something goes wrong** — post-research (verifying impact scope, checking side effects) should be a default step after execution, not just a recovery mechanism
- **Assuming reviewer output is advisory** — for critical changes (auth, payment, data migration), reviewer approval is a hard gate, not a suggestion
````

### 3c. knowledge.md

This file contains knowledge and memory management rules: when and how to update memory files, the structured project knowledge format, agent access rules for knowledge files, and skill discovery triggers.

The Boss reads this when updating project knowledge, managing memories, or discovering skills.

#### File location

```
~/.claude/protocols/knowledge.md
```

#### Content to add

````
Knowledge and memory management rules. Boss reads this when updating project knowledge, managing memories, or discovering skills.

---

## Memory Updates

WHEN a new project is encountered → record project path, tech stack, and build/test commands in `memory/projects/<name>.md` (use the Structured Knowledge Base format below)
WHEN a reusable debugging insight is gained → add to `memory/debugging.md`
WHEN the user states a preference → record in `MEMORY.md` immediately (do not defer)
WHEN a previously recorded memory is found to be wrong → correct or remove it; never leave stale data

WHEN a Large+ task starts → create `memory/active/<task-name>/` with plan.md, context.md, tasks.md (see Session Continuity in orchestration.md)

DO NOT update memory for:
- One-off tasks or session-specific context
- Unverified assumptions
- Intermediate results that may change before task completion

---

## Structured Knowledge Base

Project knowledge files live at `memory/projects/<name>.md` and must follow this exact format:

```markdown
# Project: <name>
## Quick Reference
- Path: <absolute path>
- Stack: <language, framework, key dependencies>
- Build: <build command>
- Test: <test command>
- Lint: <lint command>

## Architecture
- Entry point: <main file>
- Key directories: <dir → purpose mapping>
- Key patterns: <architectural patterns used>

## Known Issues
- <issue description> → <workaround or status>

## History
- <date>: <what was done and learned>
```

### Agent access rules

- Researchers: read project knowledge file FIRST before any codebase exploration
- Workers: read Quick Reference before starting work
- Reviewers: read Architecture and Known Issues before reviewing
- Boss: update History after each completed task

### Staleness rules

WHEN starting a new session → verify Quick Reference (build/test commands change frequently)
WHEN researcher reports unexpected structure → re-validate Architecture section
WHEN a Known Issue is resolved and verified → remove it from the list
History is append-only — never delete entries

---

## Post-Task: Skill Discovery

WHEN the same code pattern was applied to 2+ locations in the task → launch skill-discoverer (bg)
WHEN similar changes were made across 3+ files → launch skill-discoverer (bg)
WHEN the user says "this will come up again" or similar → launch skill-discoverer (bg)
WHEN a multi-step workaround was needed for a recurring problem → launch skill-discoverer (bg)

Generated skills must use `disable-model-invocation: true`.
BEFORE adding any auto-generated skill → report it to the user and get explicit confirmation first.

---

## Common Mistakes

Avoid these errors in knowledge management:

- **Writing duplicate memories** — check existing `MEMORY.md` and `memory/projects/` files before creating new entries; update in place rather than appending a second copy
- **Saving session-specific context** — do not record intermediate state, partial results, or "current task status" in permanent memory files; that belongs in `memory/in-progress.md` only
- **Not checking existing memories before creating new ones** — always read the relevant memory file first; a researcher should be launched if the file is large or its contents are uncertain
- **Updating `MEMORY.md` with unverified assumptions** — only record facts that are Confirmed (verified against code, tests, or explicit user statement); never record Uncertain or Likely findings as permanent memory
- **Not reading the project knowledge file before codebase exploration** — researchers must read `memory/projects/<name>.md` first; skipping this wastes tokens re-discovering already-known facts
- **Leaving stale Known Issues** — when an issue is fixed and verified, delete it from the Known Issues section immediately; stale entries mislead future workers and reviewers
- **Forgetting to update History** — Boss must append a History entry after every completed task on a project; this is the audit trail that enables session continuity
- **Recording preferences in the wrong file** — user preferences go in `MEMORY.md` (global), not in project files; project-specific facts go in `memory/projects/<name>.md`, not in `MEMORY.md`
````

---

## Step 4: Initialize MEMORY.md

Create a per-project memory file. This is a knowledge base that accumulates information across sessions.

### File location

```
~/.claude/projects/<project-path>/memory/MEMORY.md
```

`<project-path>` is the absolute path of your project directory converted to a hyphen-delimited string. For example, `/Users/yourname/projects/my-app` becomes `-Users-yourname-projects-my-app`.

```bash
# Example: if your project directory is /Users/yourname/projects/my-app
mkdir -p ~/.claude/projects/-Users-yourname-projects-my-app/memory
```

### Content to add

```markdown
# Auto Memory

## System Setup
- Multi-agent team system configured
- Architecture: Boss (opus) + advisor/researcher/worker/reviewer/skill-discoverer

## Discovered Patterns
(Updated as work progresses)

## User Preferences
(Updated as preferences are discovered)

## Project Notes
(Updated as projects are worked on)
```

> **Note**: Claude Code updates this file automatically. Provide the skeleton above as an initial setup, and project-specific knowledge will accumulate as you continue using the system.

---

## File Structure Summary

After completing all steps, you should have:

```
~/.claude/
  ├── settings.json                     # Step 1: permissions and feature flags
  ├── CLAUDE.md                         # Step 2: core protocol (~84 lines, loaded every turn)
  └── protocols/                        # Step 3: detailed protocols (read on demand)
      ├── orchestration.md              #   Agent dispatch, error handling, delegation
      ├── quality.md                    #   Source authority, verification, review
      └── knowledge.md                  #   Memory updates, project knowledge, skills

~/.claude/projects/<project-path>/
  └── memory/
      └── MEMORY.md                     # Step 4: per-project knowledge base
```

### What loads when

| Layer | When loaded | Size | Contains |
|-------|------------|------|----------|
| `CLAUDE.md` | Every turn (automatic) | ~84 lines / ~1050 tokens | Core rules, task sizing, agent roster, top mistakes, pointers to protocols |
| `protocols/*.md` | On demand (agent reads when needed) | ~300 lines total | Full orchestration, quality, and knowledge management rules |
| `MEMORY.md` | On demand (agents read per protocol) | Varies | Project-specific knowledge, discovered patterns, user preferences |

---

## Usage Tips

### Getting started

Start with a medium-sized task (e.g., "refactor this function" or "add tests for this module"). You will see the Boss decompose the task and delegate to subagents.

### How it works

1. When a user sends a request, the Boss evaluates the task size using the Task Sizing table in `CLAUDE.md`
2. For medium or larger tasks, the researcher gathers information first, then the advisor creates a plan (following `protocols/orchestration.md`)
3. Workers execute the plan in the background
4. For code changes, the reviewer automatically performs a quality check (following `protocols/quality.md`)
5. Knowledge is updated if applicable (following `protocols/knowledge.md`)
6. Once complete, a summary of results is reported

### Key points

- **Background execution**: Subagents run in the background, so you can send other questions or instructions via chat while work is in progress
- **Interruption and re-planning**: If requirements change mid-task, the Boss re-consults the advisor and adjusts the plan
- **Memory accumulation**: Project knowledge accumulates in MEMORY.md over sessions, making the system more efficient over time
- **Automated review**: The reviewer automatically performs quality checks on code changes without an explicit request
- **Low overhead**: The 2-layer architecture means only ~1050 tokens of protocol are loaded per turn; detailed rules are read only when the situation demands them

---

## Customization

### Editing the core CLAUDE.md

The core `CLAUDE.md` is intentionally compact. If you want to adjust high-level behavior (e.g., task sizing thresholds, agent roster), edit this file directly.

### Editing protocol files

For detailed behavior changes (e.g., error handling strategy, review rules, memory format), edit the appropriate file in `~/.claude/protocols/`:

- **orchestration.md**: Agent dispatch rules, parallel conflict prevention, error handling, session continuity, prompt guidelines, delegation patterns
- **quality.md**: Source authority hierarchy, confidence labels, verification protocol, review strategy
- **knowledge.md**: Memory update rules, project knowledge file format, skill discovery triggers

### Model selection

You can adjust the Agent Roster table in `CLAUDE.md`.

- **Cost-focused**: Assign `haiku` to more agents (e.g., use haiku for reviewer as well)
- **Quality-focused**: Lock worker (complex) to `opus`

### Concurrent agent limit

The default is **3-4 simultaneous agents** (defined in `protocols/orchestration.md`). You can increase this if your rate limits allow, but adjust based on your API usage.

### Language setting

The default setting `Respond in the same language the user uses` means the system responds in whatever language you write in. No changes are needed.

### Permissions

Customize the permissions list in `settings.json` to match your development environment.

- If you use Docker: add `"Bash(docker *)"`
- If you use Terraform: add `"Bash(terraform *)"`
- You can also add custom dangerous commands to the deny list

---

## Next Step

**Part 2: Agent Visualization Tool Setup** covers how to install a visualization tool that displays agent activity in real time on your menu bar. This adds the Signals section to `CLAUDE.md` and lets you visually monitor the agent team in action.
