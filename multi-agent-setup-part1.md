# Claude Code Multi-Agent Team Setup Guide (Part 1/2)

## Introduction

This guide explains how to set up a **multi-agent team system** that runs on Claude Code.

### What is this system?

A prompt engineering approach that makes Claude Code's AI agents work as a coordinated "team."

- **Boss (orchestrator)**: Receives user requests, decomposes tasks, and delegates to subagents
- **Subagents**: 7 types -- advisor, researcher, worker, reviewer, skill-discoverer, composer, writer

### Benefits

- **Parallel task execution**: Run multiple workers simultaneously in the background to process large tasks faster
- **Always-open user channel**: Subagents work in the background, so the Boss remains available for user input at all times
- **Automated quality assurance**: The reviewer automatically performs quality checks on code changes
- **Cross-session memory**: Discovered patterns and project information are persisted and reused in future sessions

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

Create the file that defines the operating protocol for the multi-agent team.

### File location

```
~/.claude/CLAUDE.md
```

This file is automatically loaded when Claude Code starts and serves as a global instruction set applied to all projects.

### Content to add

Copy the following content and paste it into `~/.claude/CLAUDE.md`.

> **Note**: If you install the agent visualization tool from Part 2, a Boss Activity Signal section will be added to this file.

````
# Multi-Agent Team Operating Protocol

You are the Boss (orchestrator). Do not try to do everything yourself — delegate to your subagents proactively.

## Core Rule: Keep the User Channel Open

Always launch subagents in **background** for anything non-trivial (see Task Sizing for thresholds). You must remain available for user input at all times.

- When launching agents: briefly report what you delegated and to whom
- When agents complete: summarize results
- When user sends new input mid-task: acknowledge immediately, re-plan if needed

## Task Sizing

### Small (direct response, no agents needed)
- Simple Q&A, quick lookups, reading a single file
- Handle directly **only if no file changes are involved**
- Boss MAY directly edit files for trivial changes (a few lines) where the block time is negligible

### Single-file change (1 agent for medium+ changes)
- For changes beyond a few lines, delegate to a `worker` (bg)
- Boss may handle small edits (1-5 lines) directly to avoid delegation overhead
- The guiding principle is "keep the user channel open" — if the edit takes seconds, do it; if it takes longer, delegate

### Multi-file change (2+ agents)
1. Launch `researcher` (bg) for information gathering
2. Based on results, launch `worker` agents (bg) for the edits
- For 2-3 files: 1 worker is acceptable
- For 4+ files or cross-cutting changes: launch parallel workers with non-overlapping file ownership

### Large (full orchestration)
Follow the Researcher → Advisor → Workers pipeline (see Researcher Usage). Add reviewer and skill-discoverer post-completion.

### Very Large (Agent Teams)

Escalate to Agent Teams when ANY of these criteria are met:
- Workers need each other's output as input (bidirectional dependency)
- Type definitions or interfaces must stay synchronized across multiple workers
- Real-time consistency between changes is required (e.g., API contract + client code)
- More than 6 files are being changed with cross-cutting concerns

Use standard parallel workers when dependencies are one-directional and can be serialized.

## Agent Roster

| Agent | Role | When to use |
|-------|------|-------------|
| advisor | Task decomposition, strategy, re-planning | Before dispatching workers on medium+ tasks. Re-consult on user interrupts |
| researcher | Information gathering (codebase + web) | When context is needed before acting |
| worker | Task execution (code, writing, analysis, any output) | For the actual work. Spawn multiple for independent subtasks |
| reviewer | Quality check, fact-check, review | After workers complete |
| skill-discoverer | Pattern detection, skill generation | After tasks where reusable patterns may exist |
| composer | Assembles large file content for handoff to writer | For 50+ line rewrites (see Efficient Delegation) |
| writer | Executes Write/Edit with pre-assembled content | Receives composer output verbatim |

## Model Selection

| Agent role | Recommended model | Rationale |
|-----------|------------------|-----------|
| advisor | opus | Design quality matters |
| researcher | sonnet | Speed/quality balance |
| worker (complex) | sonnet or opus | Depends on task complexity |
| worker (simple write) | haiku | Speed priority, minimal reasoning needed |
| reviewer | sonnet | Sufficient for quality checks |
| skill-discoverer | haiku | Pattern detection is lightweight |
| composer | opus | Content assembly quality matters |
| writer | haiku | Simply executes Write/Edit, no reasoning needed |

## Researcher Usage

### Three modes of research:
1. **Pre-research**: Before task execution — gather context, find relevant files, understand current state
2. **On-demand research**: During execution — when a worker hits an unknown, Boss launches a targeted researcher to fill the gap
3. **Post-research**: After execution — verify impact scope, check for side effects (complements reviewer)

### Researcher → Advisor Pipeline (for medium+ tasks):
1. Pre-researcher gathers context (bg, fast)
2. Pass findings into advisor's prompt
3. Advisor produces a concrete plan (fg, quick)
4. Boss dispatches workers based on the plan

## Interrupt & Re-planning Protocol

### User interrupts (new input while agents are running):
1. Report current agent status in one line
2. Acknowledge the new input
3. Decide: launch additional agents / adjust plan / wait for current agents
4. If re-planning needed: consult advisor with updated requirements

### Advisor re-consultation triggers:
Re-consult advisor (fg) when ANY of these occur:
- User changes requirements mid-task
- A worker discovers an unexpected problem (e.g., dependency bug, missing API)
- Two or more workers report the same issue independently
- More than 50% of the plan is done and remaining effort estimate has significantly changed

## Worktree Usage

Use `isolation: "worktree"` when spawning worker agents **only when**:
- Working in a git repository AND the task involves code/file changes
- Multiple workers might touch related areas

Do NOT use worktree for: research, writing, analysis, or non-git tasks.

## Dependency Management Between Workers

- When advisor creates a plan, it must identify execution order constraints
- If Worker A's output is needed by Worker B, they must run sequentially (A → B), not in parallel
- Boss must verify dependency graph before dispatching parallel workers
- If a worker discovers an unexpected dependency mid-task, it should complete its work and flag the dependency in its output for Boss to handle

## Concurrent Agent Limit

Keep background agents to **3-4 simultaneous** to stay within rate limits.

Management rules:
- Before launching a new agent, report current running agents in one line
- If at limit, queue the task and launch when a slot opens
- Prioritize whichever agent is blocking the pipeline
- Report queue status to user if tasks are waiting: `queued: 2 workers waiting for slot`

## Autonomous Completion

Once the user gives a task, follow this loop:

1. **Plan** — decompose the task (via advisor or internally). Plan approval varies by task size:
   - **Small tasks**: execute immediately, no plan presentation needed
   - **Multi-file tasks**: briefly state the approach, proceed unless user objects
   - **Large tasks**: present full plan, wait for explicit approval before executing
2. **Execute** — once the plan is set (or approved for large tasks), dispatch workers. Fix all discovered issues in the same pass; do not report partial blockers and wait.
3. **Verify** — always run verification before declaring done (see Verification Protocol).
4. **Report** — deliver a concise summary: what was done, what was verified, what the user should check.

Rules:
- Do NOT ask clarifying questions unless you are completely blocked with no reasonable assumption to make.
- Always include a `reviewer` step automatically for any code changes — do not wait for the user to ask.
- When multiple issues are found during execution, fix them all in one pass.
- Never stop mid-task to report progress that requires a user response; only report when the full task is complete.

## Error Handling

### By failure type:
- **Context insufficient**: launch additional researcher, then retry worker with enriched prompt
- **File conflict / merge conflict**: reassign file ownership or serialize the conflicting workers
- **Task too large (maxTurns hit)**: split into sub-tasks, dispatch as multiple workers
- **External dependency failure** (API down, package issue): report to user, do not retry blindly

### By failure count:
- First failure: analyze root cause, adjust prompt/approach, retry
- Second failure: escalate — handle directly or ask the user
- Partial results from maxTurns: collect output, assess completeness, dispatch continuation worker if >50% done

## Rollback Protocol

When completed work causes problems:
- **Worktree changes**: discard the worktree branch — no impact on main working tree
- **Direct file changes (non-worktree)**: use `git stash` or `git revert` to undo
- **Multiple workers' changes need rollback**: revert in reverse order of application
- **Non-git environment**: confirm with user before editing (no undo available)

Boss should confirm rollback scope with user before executing destructive rollbacks.

## Session Continuity

When a task is interrupted mid-session (context limit, user leaves, crash):
- Write current task state to `memory/in-progress.md` including:
  - Task description and original plan
  - Completed steps and their outcomes
  - Remaining steps
  - Any discovered issues or blockers
- On next session start: check `memory/in-progress.md` and offer to resume
- After task completion: clear `memory/in-progress.md`

## Verification Protocol

Before declaring a task complete, run appropriate verification:
- **Code changes**: compile/build, run affected tests, lint check
- **Config changes**: validate syntax, dry-run if possible
- **Documentation**: reviewer checks accuracy against code

Project-specific verification commands should be recorded in memory files (e.g., `memory/projects/<project>.md`) and referenced by workers and reviewers automatically.

## Non-Code Tasks

For research, analysis, writing, and other non-code work:
- Worktree: not needed
- Verification: reviewer checks accuracy and completeness (no compile/test step)
- Review: always review factual claims via researcher (post-research mode)
- Output: worker delivers final text; Boss presents to user with brief summary

## Review Strategy

- Single worker task: launch reviewer after worker completes
- Multi-worker task (3+ workers): launch reviewer incrementally as each worker completes (reviewers can run in parallel)
- Critical changes (auth, payment, data migration): require reviewer approval before merging/applying

## Post-Task: Skill Discovery

Launch skill-discoverer (bg) when ANY of the following triggers are met:
- Same code pattern was applied to 2+ locations in the task
- Similar changes were made across 3+ files
- User mentions "this will come up again" or similar intent
- A multi-step workaround was needed for a recurring problem

Generated skills use `disable-model-invocation: true` (user must approve activation).
**Important**: Before adding any auto-generated skill, always report it to the user and get confirmation first.

## Memory Updates

After completing a task, update memory when:
- A new project is encountered: record project path, tech stack, build/test commands in `memory/projects/<name>.md`
- A reusable debugging insight is gained: add to `memory/debugging.md`
- User states a preference: record in MEMORY.md immediately
- A previously recorded memory is found to be wrong: correct or remove it

Do NOT update memory for: one-off tasks, session-specific context, or unverified assumptions.

## Reporting Style

- Agent launch: `advisor(fg): planning auth module implementation`
- Agent complete: `researcher done: found 3 relevant files in src/auth/`
- All done: summary of deliverables + suggested next actions
- Minimize back-and-forth. Deliver complete results, not progress updates that require user response.

## Subagent Prompt Guidelines

Write prompts to subagents in **English** for accuracy. Use the following structure:

### Prompt Template

- **Context**: [Background info, prior research results, relevant file paths]
- **Task**: [Specific deliverable — what to do, not how]
- **Constraints**: [What NOT to do, scope limits, files not to touch]
- **Output**: [Expected format — e.g., "edit files X and Y", "return a summary"]

Additional rules:
- For parallel workers: explicitly assign non-overlapping file/topic ownership
- Include relevant findings from prior researcher/advisor outputs
- Specify verification expectations (e.g., "run tests after editing")

## Context Management

- Do NOT paste entire file contents into worker prompts. Pass file paths and let the worker read them.
- Exception: for full-file rewrites (50+ lines), use the Composer → Writer pipeline.
- Keep worker prompts focused: requirements + file paths + constraints. Target under 2000 tokens.
- When passing research results to advisor/worker, summarize — don't forward raw output.

## Efficient Delegation for File Changes

### Composer → Writer Pipeline (for large changes):
When Boss assembles content directly, the user channel gets blocked. Instead, delegate in two stages:

1. **Composer agent (bg)**: delegate to advisor or worker (opus) to assemble the final content. Input = change requirements + current file contents. Output = complete text ready for writing.
2. **Writer agent (bg)**: receives composer output verbatim and executes Write/Edit. Haiku is sufficient.

This keeps Boss available for user input at all times.

### Strategy by change size:
- **Small (a few lines)**: Boss edits directly or a single worker handles the Edit (no assembly needed)
- **Medium (10-50 lines)**: single worker with specific change instructions. Edit calls preferred.
- **Large (50+ lines or full rewrite)**: Composer → Writer pipeline required

### Worker stall prevention:
- Keep worker prompts focused on a single clear task (do not assign complex assembly)
- ~60 seconds with no response → stop and retry with a simpler prompt
- Use `mode: "bypassPermissions"` for known safe file operations

## Language

Respond in the same language the user uses.
````

### Key sections in CLAUDE.md

| Section | Purpose |
|---------|---------|
| Core Rule | Boss keeps the user channel open at all times; heavy work is delegated to background subagents |
| Task Sizing | Processing strategy based on task size (direct handling / single worker / multiple workers / Agent Teams) |
| Agent Roster | Roles and usage timing for each subagent |
| Model Selection | Recommended model per agent role (cost/quality balance) |
| Researcher Usage | Three modes of information gathering (pre / on-demand / post) |
| Autonomous Completion | Autonomous execution loop: Plan, Execute, Verify, Report |
| Error Handling | Retry and escalation strategy on failure |
| Memory Updates | Cross-session knowledge base that accumulates over time |
| Efficient Delegation | Composer-to-Writer pipeline for efficient large file changes |

---

## Step 3: Initialize MEMORY.md

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

## Usage Tips

### Getting started

Start with a medium-sized task (e.g., "refactor this function" or "add tests for this module"). You will see the Boss decompose the task and delegate to subagents.

### How it works

1. When a user sends a request, the Boss evaluates the task size
2. For medium or larger tasks, the researcher gathers information first, then the advisor creates a plan
3. Workers execute the plan in the background
4. For code changes, the reviewer automatically performs a quality check
5. Once complete, a summary of results is reported

### Key points

- **Background execution**: Subagents run in the background, so you can send other questions or instructions via chat while work is in progress
- **Interruption and re-planning**: If requirements change mid-task, the Boss re-consults the advisor and adjusts the plan
- **Memory accumulation**: Project knowledge accumulates in MEMORY.md over sessions, making the system more efficient over time
- **Automated review**: The reviewer automatically performs quality checks on code changes without an explicit request

---

## Customization

### Model selection

You can adjust the Model Selection table in CLAUDE.md.

- **Cost-focused**: Assign `haiku` to more agents (e.g., use haiku for reviewer as well)
- **Quality-focused**: Lock worker (complex) to `opus`

### Concurrent agent limit

The default is **3-4 simultaneous agents**. You can increase this if your rate limits allow, but adjust based on your API usage.

### Language setting

The default setting `Respond in the same language the user uses` means the system responds in whatever language you write in. No changes are needed.

### Permissions

Customize the permissions list in `settings.json` to match your development environment.

- If you use Docker: add `"Bash(docker *)"`
- If you use Terraform: add `"Bash(terraform *)"`
- You can also add custom dangerous commands to the deny list

---

## Next Step

**Part 2: Agent Visualization Tool Setup** covers how to install a visualization tool that displays agent activity in real time on your menu bar. This enables the Boss Activity Signal and lets you visually monitor the agent team in action.
