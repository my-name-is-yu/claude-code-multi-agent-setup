# Claude Code Multi-Agent Team Setup

**A copy-paste setup guide for turning Claude Code into a multi-agent orchestration system with real-time macOS menu bar visualization.**

> **Platform note:** The multi-agent protocol (Part 1) works on macOS, Linux, and WSL. The menu bar visualization tool (Part 2) requires **macOS only**.

---

## What Is This?

This repository provides ready-to-use configuration prompts and source code for setting up Claude Code as a **multi-agent team system**. No framework or external dependencies beyond Claude Code itself -- just prompt engineering and a lightweight visualization layer.

The setup is split into two parts:

| Part | File | Description |
|------|------|-------------|
| 1 | [multi-agent-setup-part1.md](multi-agent-setup-part1.md) | Multi-agent team protocol (CLAUDE.md + settings.json) |
| 2 | [multi-agent-setup-part2.md](multi-agent-setup-part2.md) | macOS menu bar visualization tool (source code + installation) |

### What it does

- **Boss (orchestrator)** automatically decomposes tasks and delegates to specialized subagents running in the background
- **7 agent roles** -- advisor, researcher, worker, reviewer, skill-discoverer, composer, writer -- each with a defined purpose and recommended model
- **Keeps the user channel open** -- subagents run in the background so the Boss always remains responsive to user input
- **Auto-verification and quality review** -- a reviewer agent is automatically launched after code changes; no manual request needed
- **Session memory** -- patterns, project notes, and debugging insights persist across sessions via MEMORY.md
- **macOS menu bar app** shows real-time agent status, token usage, and estimated cost via Server-Sent Events (SSE)

<!-- TODO: Add screenshot of menu bar visualization -->

---

## Prerequisites

- **Claude Code CLI** installed and authenticated
- **Claude Max plan** or an API key with access to Opus / Sonnet / Haiku models
- **macOS 13+ (Ventura or later)** -- required for the menu bar visualization tool (Part 2)
- **Node.js v18+** -- for the visualization server
- **Xcode Command Line Tools** -- install with `xcode-select --install` if not present

> *Part 1 (the protocol itself) does not require macOS. It works anywhere Claude Code runs.*

---

## Quick Start

### Step 1: Set up the multi-agent protocol

Follow **[multi-agent-setup-part1.md](multi-agent-setup-part1.md)** to configure:

1. `~/.claude/settings.json` -- permissions, environment variables, Agent Teams flag
2. `~/.claude/CLAUDE.md` -- the full multi-agent operating protocol
3. `~/.claude/projects/<path>/memory/MEMORY.md` -- initial memory template

### Step 2: Install the visualization tool (macOS only)

Follow **[multi-agent-setup-part2.md](multi-agent-setup-part2.md)** to set up:

1. Visualization server and Claude Code hooks
2. Native macOS menu bar app (compiled from Swift)
3. LaunchAgent for auto-start on login

### Step 3: Try it out

Open Claude Code and give it a moderately complex task -- for example, "refactor this module and add tests." The Boss will decompose the task, delegate to subagents, and you will see live status updates in the menu bar.

---

## Architecture Overview

The system has two layers: the **agent protocol** (prompt-driven, runs inside Claude Code) and the **visualization pipeline** (external server + native macOS app).

```
User
  |
  v
Boss (Opus) -----> Subagents (background)
  |                  |-- advisor    (task decomposition, strategy)
  |                  |-- researcher (information gathering)
  |                  |-- worker     (code, writing, analysis)
  |                  |-- reviewer   (quality checks)
  |                  |-- skill-discoverer (pattern detection)
  |                  |-- composer   (large content assembly)
  |                  +-- writer     (file writes)
  |
  | Claude Code Hooks (on subagent start/stop)
  v
Visualization Server (Node.js, port 1217)
  |
  | SSE (Server-Sent Events)
  v
macOS Menu Bar App (Swift / Cocoa)
  |-- Agent status (running / done / idle)
  |-- Token usage + estimated cost
  +-- macOS notifications on completion/error
```

The hooks fire automatically when Claude Code spawns or completes a subagent. The server aggregates state and pushes updates to the menu bar app via SSE.

---

## What's in Each File

### Part 1: `multi-agent-setup-part1.md`

| Component | Description |
|-----------|-------------|
| `settings.json` config | Permissions allow-list/deny-list, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` flag |
| `CLAUDE.md` protocol | Full operating protocol -- task sizing, agent roster, model selection, researcher pipeline, interrupt handling, error recovery, memory management, efficient delegation |
| `MEMORY.md` template | Initial skeleton for session-persistent memory (patterns, preferences, project notes) |

### Part 2: `multi-agent-setup-part2.md`

Setup guide for the [agent-visualization](https://github.com/my-name-is-yu/agent-visualization) tool. Covers cloning, installation, hooks configuration, and CLAUDE.md signal setup. The tool source code lives in its own repository.

---

## Customization

### Model selection per agent role

Edit the **Model Selection** table in `~/.claude/CLAUDE.md`:

| Agent role | Default model | Alternatives |
|-----------|--------------|-------------|
| advisor | opus | sonnet (faster, lower cost) |
| researcher | sonnet | haiku (faster) |
| worker (complex) | sonnet or opus | -- |
| worker (simple write) | haiku | -- |
| reviewer | sonnet | haiku (lower cost) |
| composer | opus | sonnet |
| writer | haiku | -- |

### Environment variables (visualization)

Set these in `~/.claude/settings.json` under `env`, or in the LaunchAgent plist:

| Variable | Description | Default |
|----------|-------------|---------|
| `AGENT_VIZ_PORT` | Server port | `1217` |
| `AGENT_VIZ_COST_PER_MTOK` | Cost per million tokens (USD) | `9` |
| `AGENT_VIZ_BOSS_MODEL` | Boss model display name | `opus` |
| `AGENT_VIZ_AUTO_RESET_SECONDS` | Seconds after all agents complete before state resets | `60` |
| `AGENT_VIZ_CLEANUP_MINUTES` | Minutes to retain completed/errored agent info | `30` |

> If you change the port, also update the heartbeat URL in `~/.claude/CLAUDE.md`.

### Concurrent agent limit

The default is **3-4 simultaneous** background agents. Adjust the "Concurrent Agent Limit" section in `CLAUDE.md` if your rate limits allow more.

### Permissions

The `settings.json` permissions list is a recommended starting point. Add commands for your toolchain (e.g., `"Bash(docker *)"`, `"Bash(terraform *)"`) or tighten the deny-list as needed.

---

## Troubleshooting

For detailed troubleshooting steps, see the **troubleshooting section** in [multi-agent-setup-part2.md](multi-agent-setup-part2.md#トラブルシューティング).

Common checks:

- **Logs**: `~/Library/Logs/agent-visualization/` -- both `server.log` and `menubar.log`
- **Server not connected**: verify the server is running with `curl http://localhost:1217/state`
- **Menu bar icon missing**: check if the app process is running with `pgrep -f AgentMenuBar`
- **Swift compile error**: ensure Xcode Command Line Tools are up to date (`xcode-select --install`)
- **Restart server**: `launchctl kickstart -k gui/$(id -u)/com.agent-visualization.server`
- **Restart menu bar app**: `launchctl kickstart -k gui/$(id -u)/com.agent-visualization.menubar`

---

## License

MIT

---

## Credits

- Built for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) by [Anthropic](https://www.anthropic.com/)
- macOS menu bar app written in Swift using the Cocoa framework
- Visualization server powered by [Express](https://expressjs.com/) (Node.js)

