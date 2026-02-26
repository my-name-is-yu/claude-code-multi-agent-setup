# Part 2: Agent Visualization Tool (macOS Menu Bar)

## Overview

Real-time macOS menu bar app that visualizes Claude Code's multi-agent activity. Shows agent status, output, token usage, and estimated cost via Server-Sent Events. A persistent icon in the menu bar gives you an at-a-glance view of the Boss and all subagents (running / done / idle), with click-through detail views, session usage tracking, macOS notifications on completion, and auto-reset after all agents finish.

---

## Prerequisites

- macOS 13+ (Ventura or later)
- Node.js v18+
- Xcode Command Line Tools (`xcode-select --install`)
- Claude Code CLI installed and authenticated
- Part 1 (Multi-Agent Team Setup) completed

---

## Step 1: Clone and Install

```bash
git clone https://github.com/my-name-is-yu/agent-visualization.git ~/agent-visualization
cd ~/agent-visualization
./install.sh
```

The installer handles: npm dependencies, Swift compilation, app bundle creation, LaunchAgent registration, and auto-start configuration.

---

## Step 2: Configure Claude Code Hooks

Add the following to `~/.claude/settings.json` under `"hooks"`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Task",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase pre", "async": true }]
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase pre", "async": true }]
      },
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase pre", "async": true }]
      },
      {
        "matcher": "Write",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase pre", "async": true }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase post", "async": true }]
      },
      {
        "matcher": "TaskOutput",
        "hooks": [{ "type": "command", "command": "node ~/agent-visualization/hook.js --phase post", "async": true }]
      }
    ]
  }
}
```

> Note: Replace `~/agent-visualization` with the actual path if you cloned to a different location.

---

## Step 3: Add Visualization Signals to CLAUDE.md

Add the following sections to your `~/.claude/CLAUDE.md` (the multi-agent protocol from Part 1):

### Boss Activity Signal

~~~~
## Boss Activity Signal

When you start processing a user request, signal your activity to the menu bar visualization by running:
```
curl -sX POST http://127.0.0.1:1217/heartbeat > /dev/null 2>&1
```
Run this ONCE at the beginning of each user request (not on every tool call). This keeps the menu bar status in sync with Boss activity. Do not mention this to the user.
~~~~

### Background Agent Completion Signal

~~~~
## Background Agent Completion Signal

When a background agent completes (you receive a task-notification), signal its completion to the visualization server:
```
curl -sX POST http://127.0.0.1:1217/complete -H 'Content-Type: application/json' -d '{"description":"<description>","result":"<first 200 chars of result>","tokens":<total_tokens>,"tool_uses":<tool_uses>,"duration_ms":<duration_ms>}' > /dev/null 2>&1
```
Extract the values from the `<task-notification>` block (`<summary>` for description, `<result>` for result, `<usage>` for tokens/tool_uses/duration_ms). If the task failed, add `"is_error":true`. Run this silently without mentioning it to the user. Always run this for EVERY background agent completion — this is the only way the menu bar learns an agent has finished.
~~~~

---

## Step 4: Verify Installation

```bash
# Check server is running
curl http://localhost:1217/state

# Check menu bar app is running
pgrep -f AgentMenuBar

# Test: open Claude Code and run a multi-agent task
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Server not responding | `launchctl kickstart -k gui/$(id -u)/com.agent-visualization` |
| Menu bar icon missing | `launchctl kickstart -k gui/$(id -u)/com.agent-visualization.menubar` |
| Agent status stuck on "running" | Check hooks are configured in settings.json; check server logs at `~/Library/Logs/agent-visualization/` |
| Swift compile error | Run `xcode-select --install` to update Command Line Tools |
| Agents complete but no output/usage shown | Ensure "Background Agent Completion Signal" section is in your CLAUDE.md |

---

## Customization

See environment variables in the [agent-visualization README](https://github.com/my-name-is-yu/agent-visualization#environment-variables) for configuration options (port, cost rates, auto-reset timeout, etc.).

---

## Uninstall

```bash
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.plist
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.menubar.plist
rm ~/Library/LaunchAgents/com.agent-visualization*.plist
rm -rf ~/agent-visualization
```
