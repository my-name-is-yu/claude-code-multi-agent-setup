# Part 2: Setting Up the macOS Menu Bar Visualization Tool

> **Multi-Agent Team Visualization Tool** -- A tool for monitoring Claude Code's multi-agent operations in real time from the macOS menu bar.

---

## What This Visualization Tool Can Do

- **Real-time agent status in the macOS menu bar** -- A persistent icon appears in the menu bar, giving you an at-a-glance view of the current status (running / done / idle) for the Boss and all agents.
- **Agent detail view** -- Click on any agent to see detailed information including its prompt, output preview, execution time, and token count.
- **Session usage tracking** -- Token counts and estimated cost (USD) are aggregated in real time.
- **macOS notifications** -- Receive native macOS Notification Center alerts when an agent completes or encounters an error.
- **Auto-reset** -- State is automatically cleared 60 seconds after all agents finish, preparing for the next task.

## Prerequisites

- **macOS 13 (Ventura) or later**
- **Node.js** must be installed (verify with `node -v`)
- **Xcode Command Line Tools** must be installed
  - If not installed: `xcode-select --install`
- **Part 1 (Multi-Agent Team Setup) must be completed**

---

## Step 1: Create the Project Directory

Run the following in Terminal to create the project directory structure.

```bash
mkdir -p ~/agent-visualization/menubar/AgentMenuBar.app/Contents/MacOS
cd ~/agent-visualization
```

---

## Step 2: Create the Files

Create each of the following files. Copy the contents of each code block exactly and save them at the specified paths.

### 2.1 package.json

**Path:** `~/agent-visualization/package.json`

```json
{
  "name": "agent-visualization",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

### 2.2 server.js

**Path:** `~/agent-visualization/server.js`

The main visualization server. It manages agent state and provides data to the menu bar app.

```javascript
const fs = require('fs');
const path = require('path');
const express = require('express');
const crypto = require('crypto');

const PORT = process.env.PORT || 1217;

// Approximate blended cost ~$9/MTok (configurable via env)
const COST_PER_TOKEN = parseFloat(process.env.AGENT_VIZ_COST_PER_MTOK || '9') / 1_000_000;
const BOSS_MODEL = process.env.AGENT_VIZ_BOSS_MODEL || 'opus';
const BOSS_ACTIVE_MS = 30_000;

const app = express();
app.use(express.json());

// In-memory agent state
const agents = new Map(); // key -> AgentRecord

// Auto-reset timer: clear state after all agents complete
const AUTO_RESET_MS = parseInt(process.env.AGENT_VIZ_AUTO_RESET_SECONDS || '60', 10) * 1000;
let autoResetTimer = null;
let lastCompletionTime = 0; // timestamp of last agent completion
let lastEventTime = 0; // timestamp of last hook event (tracks Boss activity)

// Ring-buffer for messages (max 200 entries)
const MAX_MESSAGES = 200;
const messages = [];
let messageCounter = 0;

// SSE clients for push notifications
const sseClients = new Set();

// Session usage tracking
const sessionUsage = {
  total_tokens: 0,
  tool_uses: 0,
  duration_ms: 0,
  agent_count: 0,
};

// State persistence
const STATE_FILE = process.env.AGENT_VIZ_STATE_FILE
  || path.join(process.env.HOME || '/tmp', '.agent-visualization-state.json');

let lastSaveTime = 0;
function saveState() {
  try {
    const data = JSON.stringify({
      agents: [...agents.entries()],
      messages,
      messageCounter,
      sessionUsage,
      lastCompletionTime,
    });
    const tmpFile = STATE_FILE + '.tmp';
    fs.writeFileSync(tmpFile, data);
    fs.renameSync(tmpFile, STATE_FILE);
    lastSaveTime = Date.now();
  } catch (e) {
    console.error('[saveState] Failed:', e.message);
  }
}

function loadState() {
  try {
    if (!fs.existsSync(STATE_FILE)) return;
    const raw = fs.readFileSync(STATE_FILE, 'utf8');
    const data = JSON.parse(raw);
    if (data.agents) {
      for (const [key, record] of data.agents) {
        agents.set(key, record);
      }
    }
    if (data.messages) {
      messages.length = 0;
      messages.push(...data.messages);
    }
    if (data.messageCounter) messageCounter = data.messageCounter;
    if (data.sessionUsage) Object.assign(sessionUsage, data.sessionUsage);
    // Don't restore lastCompletionTime — server restart means old grace period is irrelevant
    lastCompletionTime = 0;
    // Fix 1: Clear stale running agents from a previous crash
    const now = new Date().toISOString();
    let staleCount = 0;
    for (const record of agents.values()) {
      if (record.status === 'running') {
        record.status = 'errored';
        record.error = 'Server restarted while agent was running';
        record.ended_at = now;
        staleCount++;
      }
    }
    if (staleCount > 0) {
      console.log(`[loadState] Marked ${staleCount} stale running agent(s) as errored`);
    }
    console.log(`[loadState] Loaded ${agents.size} agents from ${STATE_FILE}`);
    // Schedule auto-reset if all agents are done (no running agents)
    if (agents.size > 0) {
      scheduleAutoReset();
    }
  } catch (e) {
    console.error('[loadState] Failed:', e.message);
  }
}

function parseUsage(output) {
  if (typeof output !== 'string') return null;
  // Try <usage>...</usage> block format (Claude Code TaskOutput)
  const usageBlock = output.match(/<usage>([\s\S]*?)<\/usage>/);
  const text = usageBlock ? usageBlock[1] : output;
  const totalMatch = text.match(/total_tokens[:\s]+(\d+)/);
  const toolMatch = text.match(/tool_uses[:\s]+(\d+)/);
  const durMatch = text.match(/duration_ms[:\s]+(\d+)/);
  if (!totalMatch && !toolMatch && !durMatch) return null;
  return {
    total_tokens: totalMatch ? parseInt(totalMatch[1], 10) : 0,
    tool_uses: toolMatch ? parseInt(toolMatch[1], 10) : 0,
    duration_ms: durMatch ? parseInt(durMatch[1], 10) : 0,
  };
}

function makeKey(session_id, description) {
  return crypto
    .createHash('sha1')
    .update(`${session_id}:${description}`)
    .digest('hex')
    .slice(0, 12);
}

function isError(is_error, tool_output) {
  if (is_error === true) return true;
  if (is_error === false) return false;
  if (typeof tool_output === 'string') {
    const sample = tool_output.slice(0, 500).toLowerCase();
    return /\berror[:;\s]|\bfailed\b|\bexception\b|\btraceback\b/.test(sample);
  }
  return false;
}

// Find the deepest currently-running agent for a given session_id.
function findDeepestRunningAgent(session_id, excludeKey) {
  let best = null;
  for (const record of agents.values()) {
    if (record.id === excludeKey) continue;
    if (record.session_id !== session_id) continue;
    if (record.status !== 'running') continue;
    if (!best || new Date(record.started_at) > new Date(best.started_at)) {
      best = record;
    }
  }
  return best;
}

function addMessage(from_id, to_id, type) {
  const entry = {
    id: String(++messageCounter),
    from_id,
    to_id,
    type,
    timestamp: new Date().toISOString(),
  };
  messages.push(entry);
  if (messages.length > MAX_MESSAGES) {
    messages.splice(0, messages.length - MAX_MESSAGES);
  }
}

function buildTasks() {
  return Array.from(agents.values()).map((a) => ({
    id: a.id,
    name: a.description,
    status: a.status,
    subagent_type: a.subagent_type,
  }));
}

function buildSessions() {
  const map = new Map();
  for (const a of agents.values()) {
    const sid = a.session_id || 'unknown';
    if (!map.has(sid)) map.set(sid, { session_id: sid, agent_count: 0, running: 0, completed: 0, errored: 0 });
    const s = map.get(sid);
    s.agent_count++;
    if (a.status === 'running') s.running++;
    else if (a.status === 'completed') s.completed++;
    else if (a.status === 'errored') s.errored++;
  }
  return Array.from(map.values());
}

function buildState() {
  const allAgents = Array.from(agents.values());
  const summary = {
    total: allAgents.length,
    running: allAgents.filter((a) => a.status === 'running').length,
    completed: allAgents.filter((a) => a.status === 'completed').length,
    errored: allAgents.filter((a) => a.status === 'errored').length,
  };

  const list = allAgents
    .sort((a, b) => new Date(b.started_at) - new Date(a.started_at))
    .slice(0, 200);

  // Boss + agent combined status
  const hasRunningAgents = summary.running > 0;
  const bossActive = (Date.now() - lastEventTime) < BOSS_ACTIVE_MS;
  let bossStatus;
  if (hasRunningAgents) {
    bossStatus = 'running';  // agents still active → running
  } else if (bossActive) {
    bossStatus = 'running';  // boss recently active (heartbeat/hook) → running
  } else if (allAgents.length > 0) {
    bossStatus = 'done';     // all agents finished, boss inactive → done
  } else {
    bossStatus = 'idle';     // nothing happening → idle
  }

  return {
    type: 'state',
    summary,
    boss: { status: bossStatus, model: BOSS_MODEL },
    agents: list,
    messages: messages.slice(),
    tasks: buildTasks(),
    sessions: buildSessions(),
    usage: { ...sessionUsage, estimated_cost_usd: Math.round(sessionUsage.total_tokens * COST_PER_TOKEN * 10000) / 10000, usage_available: sessionUsage.total_tokens > 0 },
  };
}

function notifyClients() {
  const data = JSON.stringify({ type: 'state-changed', timestamp: Date.now() });
  for (const client of sseClients) {
    try {
      client.write(`data: ${data}\n\n`);
    } catch (e) {
      sseClients.delete(client);
    }
  }
}

// ── Routes ───────────────────────────────────────────────────────────────────

// GET /state - return current state for polling clients (menu bar app)
app.get('/state', (req, res) => {
  const state = buildState();
  res.json(state);
});

// POST /heartbeat - lightweight Boss activity signal
app.post('/heartbeat', (req, res) => {
  lastEventTime = Date.now();
  if (req.body && req.body.model) {
    // Allow runtime model override (not implemented yet, reserved)
  }
  notifyClients();
  res.json({ ok: true });
});

// GET /events - SSE stream for state change notifications
app.get('/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });
  res.write('\n');
  sseClients.add(res);
  req.on('close', () => {
    sseClients.delete(res);
  });
});

// SSE keep-alive: send comment ping every 20 seconds
setInterval(() => {
  for (const client of sseClients) {
    try {
      client.write(':keepalive\n\n');
    } catch (e) {
      sseClients.delete(client);
    }
  }
}, 20_000);

// POST /event - receive hook events from Claude Code
app.post('/event', (req, res) => {
  const body = req.body;

  lastEventTime = Date.now();
  const session_id = body.session_id || '';
  const hook_phase = body.hook_phase;
  const tool_name = body.tool_name || '';
  const tool_input = body.tool_input || {};
  const tool_output = body.tool_output || null;
  const is_error = body.is_error;

  // Non-agent tool calls (Bash, Edit, Write, etc.) — just update lastEventTime
  if (tool_name !== 'Task' && tool_name !== 'TaskOutput') {
    notifyClients();
    res.json({ ok: true });
    return;
  }

  // Handle TaskOutput events — marks background agents as completed
  if (tool_name === 'TaskOutput' && hook_phase === 'post') {
    const taskId = tool_input.task_id || '';
    // Find a running background agent: match by agentId first, fallback to most recent running bg agent
    let matchedRecord = null;
    for (const record of agents.values()) {
      if (record.status === 'running' && record.background && record.agentId === taskId) {
        matchedRecord = record;
        break;
      }
    }
    if (!matchedRecord) {
      // Fallback: pick the oldest running background agent in the same session
      let oldest = null;
      for (const record of agents.values()) {
        if (record.status === 'running' && record.background && record.session_id === session_id) {
          if (!oldest || new Date(record.started_at) < new Date(oldest.started_at)) {
            oldest = record;
          }
        }
      }
      matchedRecord = oldest;
    }
    if (matchedRecord) {
      const record = matchedRecord;
      record.last_activity = new Date().toISOString();
      const ended_at = new Date();
      const errored = isError(is_error, tool_output);
      record.status = errored ? 'errored' : 'completed';
      record.ended_at = ended_at.toISOString();
      record.duration_ms = ended_at - new Date(record.started_at);
      lastCompletionTime = Date.now();
      record.error = errored && typeof tool_output === 'string'
        ? tool_output.slice(0, 300) : null;
      record.output_preview = typeof tool_output === 'string'
        ? tool_output.slice(0, 800) : null;
      const usage = parseUsage(tool_output);
      if (usage) {
        record.usage = usage;
      }
      if (usage && usage.duration_ms) record.duration_ms = usage.duration_ms;
      const to_id = record.parent_id || '__user__';
      addMessage(record.id, to_id, 'Response');
      sessionUsage.agent_count++;
      if (usage) {
        if (usage.total_tokens) sessionUsage.total_tokens += usage.total_tokens;
        if (usage.tool_uses) sessionUsage.tool_uses += usage.tool_uses;
        if (usage.duration_ms) sessionUsage.duration_ms += usage.duration_ms;
      }
      scheduleAutoReset();
    }
    notifyClients();
    res.json({ ok: true });
    if (Date.now() - lastSaveTime > 5000) saveState();
    return;
  }

  const description = tool_input.description || body.tool_name || '';
  const key = body.tool_use_id || makeKey(session_id, description);

  if (hook_phase === 'pre') {
    cancelAutoReset();

    // If no agents are currently running AND enough time has passed since the last
    // completion, this is a new batch — reset state. The 60s grace period prevents
    // resetting between sequential agent spawns from the same Boss session.
    let hasRunning = false;
    for (const record of agents.values()) {
      if (record.status === 'running') { hasRunning = true; break; }
    }
    const timeSinceLastCompletion = Date.now() - lastCompletionTime;
    if (!hasRunning && agents.size > 0 && timeSinceLastCompletion > 60_000) {
      resetState();
    }

    const parentAgent = findDeepestRunningAgent(session_id, key);
    const parent_id = parentAgent ? parentAgent.id : '__user__';

    const record = {
      id: key,
      session_id,
      description,
      prompt: typeof tool_input.prompt === 'string'
        ? tool_input.prompt.slice(0, 1500)
        : '',
      subagent_type: tool_input.subagent_type || 'unknown',
      background: Boolean(tool_input.run_in_background),
      status: 'running',
      started_at: new Date().toISOString(),
      last_activity: new Date().toISOString(),
      ended_at: null,
      duration_ms: null,
      error: null,
      output_preview: null,
      output_file: tool_input.output_file || null,
      parent_id,
      usage: null,  // { total_tokens, tool_uses, duration_ms }
    };
    agents.set(key, record);

    const msgType = parent_id === '__user__' ? 'Prompt' : 'TaskCreate';
    addMessage(parent_id, key, msgType);
  }

  if (hook_phase === 'post') {
    // Try to find record by key first; if not found, search by session+description+running
    let record = agents.get(key);
    if (!record) {
      for (const [existingKey, r] of agents.entries()) {
        // Match running agents, or errored agents from a server restart (not genuinely errored)
        const isMatchable = r.status === 'running' ||
          (r.status === 'errored' && r.error === 'Server restarted while agent was running');
        if (r.session_id === session_id && r.description === description && isMatchable) {
          record = r;
          // Re-key: delete old key and set under new key
          agents.delete(existingKey);
          record.id = key;
          agents.set(key, record);
          break;
        }
      }
    }
    if (!record) {
      // Post with no matching pre — create a minimal record (duration unknown)
      record = {
        id: key,
        session_id,
        description,
        prompt: '',
        subagent_type: tool_input.subagent_type || 'unknown',
        background: Boolean(tool_input.run_in_background),
        status: 'running',
        started_at: new Date().toISOString(),
        last_activity: new Date().toISOString(),
        ended_at: null,
        duration_ms: null,
        error: null,
        output_preview: null,
        output_file: null,
        parent_id: '__user__',
        usage: null,  // { total_tokens, tool_uses, duration_ms }
      };
      agents.set(key, record);
    }

    // Update last_activity whenever the post hook fires for this agent
    record.last_activity = new Date().toISOString();

    // Detect background agent launch: post fires immediately but agent is still running.
    // For background tasks, tool_output may be null/undefined (not yet available) or
    // a string containing "Async agent launched".
    const isBgLaunch = record.background &&
      (tool_output == null ||
       (typeof tool_output === 'string' && /Async agent launched/i.test(tool_output)));

    if (isBgLaunch) {
      // Extract output_file and agentId but keep agent as "running"
      if (typeof tool_output === 'string') {
        const outputFileMatch = tool_output.match(/output_file:\s*(\S+)/);
        if (outputFileMatch) record.output_file = outputFileMatch[1];
        const agentIdMatch = tool_output.match(/agentId:\s*(\S+)/);
        if (agentIdMatch) record.agentId = agentIdMatch[1];
      }
      // Don't mark as completed — agent is still running in background
    } else {
      // Now update the record
      const ended_at = new Date();
      const errored = isError(is_error, tool_output);

      record.status = errored ? 'errored' : 'completed';
      record.ended_at = ended_at.toISOString();
      record.duration_ms = ended_at - new Date(record.started_at);
      lastCompletionTime = Date.now();
      record.error = errored && typeof tool_output === 'string'
        ? tool_output.slice(0, 300)
        : null;
      record.output_preview = typeof tool_output === 'string'
        ? tool_output.slice(0, 800)
        : null;

      // Extract output_file path from tool_output
      const outputFileMatch = typeof tool_output === 'string' && tool_output.match(/output_file:\s*(\S+)/);
      if (outputFileMatch) {
        record.output_file = outputFileMatch[1];
      }

      // If duration is suspiciously short (under 1s) and usage has duration, use that instead
      const usage = parseUsage(tool_output);
      if (usage) {
        record.usage = usage;
      }
      if (record.duration_ms < 1000 && usage && usage.duration_ms) {
        record.duration_ms = usage.duration_ms;
      }

      const to_id = record.parent_id || '__user__';
      addMessage(key, to_id, 'Response');

      // Always count the agent; only add token data when available
      sessionUsage.agent_count++;
      if (usage) {
        if (usage.total_tokens) sessionUsage.total_tokens += usage.total_tokens;
        if (usage.tool_uses) sessionUsage.tool_uses += usage.tool_uses;
        if (usage.duration_ms) sessionUsage.duration_ms += usage.duration_ms;
      }
    }

    scheduleAutoReset(); // Check if all agents are now done
  }

  notifyClients();
  res.json({ ok: true });
  if (Date.now() - lastSaveTime > 5000) saveState();
});

function resetState() {
  agents.clear();
  messages.length = 0;
  messageCounter = 0;
  sessionUsage.total_tokens = 0;
  sessionUsage.tool_uses = 0;
  sessionUsage.duration_ms = 0;
  sessionUsage.agent_count = 0;
  lastEventTime = 0;
  saveState();
  notifyClients();
  console.log('[resetState] State cleared');
}

function scheduleAutoReset() {
  if (autoResetTimer) clearTimeout(autoResetTimer);
  autoResetTimer = null;

  // Check if any agents are still running
  for (const record of agents.values()) {
    if (record.status === 'running') return; // still active, don't schedule
  }

  // No running agents — schedule reset
  if (agents.size > 0) {
    console.log(`[autoReset] All agents done. Resetting in ${AUTO_RESET_MS / 1000}s`);
    autoResetTimer = setTimeout(() => {
      // Double-check no new agents started
      for (const record of agents.values()) {
        if (record.status === 'running') {
          console.log('[autoReset] Cancelled — new agent started');
          autoResetTimer = null;
          return;
        }
      }
      const bossStillActive = (Date.now() - lastEventTime) < BOSS_ACTIVE_MS;
      if (bossStillActive) {
        // Boss is still active, reschedule
        scheduleAutoReset();
        return;
      }
      resetState();
      autoResetTimer = null;
    }, AUTO_RESET_MS);
  }
}

function cancelAutoReset() {
  if (autoResetTimer) {
    clearTimeout(autoResetTimer);
    autoResetTimer = null;
    console.log('[autoReset] Cancelled — new agent started');
  }
}

// POST /reset - clear all state
app.post('/reset', (req, res) => {
  resetState();
  res.json({ ok: true, message: 'State cleared' });
});

// Cleanup: every 60s remove completed/errored agents older than configured time
const CLEANUP_MS = parseInt(process.env.AGENT_VIZ_CLEANUP_MINUTES || '30', 10) * 60 * 1000;

setInterval(() => {
  const now = Date.now();

  // Mark stale "running" agents as errored (no activity for 5+ minutes)
  const STALE_MS = 5 * 60 * 1000;
  let markedStale = false;
  for (const record of agents.values()) {
    if (record.status !== 'running') continue;
    const lastAct = new Date(record.last_activity || record.started_at);
    if (now - lastAct > STALE_MS) {
      record.status = 'errored';
      record.error = 'Agent appears stale (no activity for 5+ minutes)';
      record.ended_at = new Date().toISOString();
      record.duration_ms = now - new Date(record.started_at).getTime();
      markedStale = true;
    }
  }
  if (markedStale) {
    notifyClients();
    scheduleAutoReset();
  }

  for (const [key, record] of agents.entries()) {
    if (record.status === 'running') continue;
    const endedAt = record.ended_at ? new Date(record.ended_at).getTime() : 0;
    if (now - endedAt > CLEANUP_MS) {
      agents.delete(key);
    }
  }

  const cutoff = new Date(now - CLEANUP_MS).toISOString();
  while (messages.length > 0 && messages[0].timestamp < cutoff) {
    messages.shift();
  }
}, 60 * 1000);

loadState();

app.listen(PORT, '127.0.0.1', () => {
  console.log(`Agent visualization server running on http://localhost:${PORT}`);
});

setInterval(saveState, 30_000);

process.on('SIGTERM', () => { saveState(); process.exit(0); });
process.on('SIGINT', () => { saveState(); process.exit(0); });
```

### 2.3 hook.js

**Path:** `~/agent-visualization/hook.js`

A script invoked by Claude Code hooks. Each time Claude Code uses a tool, it receives event data from stdin and forwards it to the server.

```javascript
#!/usr/bin/env node

const http = require("http");

// Parse --phase argument, default to "post"
let phase = "post";
const args = process.argv.slice(2);
for (let i = 0; i < args.length; i++) {
  if (args[i] === "--phase" && args[i + 1]) {
    phase = args[i + 1];
    break;
  }
}

// Exit after 500ms max, no matter what
const timeout = setTimeout(() => {
  process.exit(0);
}, 500);
timeout.unref();

// Read all stdin
let raw = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  raw += chunk;
});

process.stdin.on("end", () => {
  try {
    const data = JSON.parse(raw);
    data.hook_phase = phase;
    const body = JSON.stringify(data);

    const req = http.request(
      {
        hostname: "127.0.0.1",
        port: parseInt(process.env.AGENT_VIZ_PORT || "1217", 10),
        path: "/event",
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Content-Length": Buffer.byteLength(body),
        },
      },
      () => {
        process.exit(0);
      }
    );

    req.on("error", () => {
      process.exit(0);
    });

    req.write(body);
    req.end();
  } catch (_) {
    process.exit(0);
  }
});

process.stdin.on("error", () => {
  process.exit(0);
});
```

### 2.4 AgentMenuBar.swift

**Path:** `~/agent-visualization/menubar/AgentMenuBar.swift`

Source code for the native macOS menu bar app.

```swift
import Cocoa
import Foundation
import UserNotifications

// MARK: - Config

let kMenuWidth: CGFloat = 480
let kPadding: CGFloat = 16
let kContentWidth: CGFloat = kMenuWidth - kPadding * 2

// MARK: - Data Models

struct AgentInfo {
    let id: String
    let description: String
    let subagentType: String
    let status: String
    let parentId: String?
    let durationMs: Int?
    let sessionId: String
    let outputPreview: String?
    let outputFile: String?
    let prompt: String?
    let background: Bool
    let startedAt: String
    let totalTokens: Int?
    let toolUses: Int?
}

struct SessionUsage {
    var totalTokens: Int = 0
    var toolUses: Int = 0
    var durationMs: Int = 0
    var agentCount: Int = 0
    var estimatedCostUsd: Double = 0
}

struct SessionInfo {
    let sessionId: String
    let agentCount: Int
    let running: Int
    let completed: Int
    let errored: Int
}

struct AgentState {
    var total: Int = 0
    var running: Int = 0
    var completed: Int = 0
    var errored: Int = 0
    var bossStatus: String = "idle"
    var bossModel: String = "opus"
    var agents: [AgentInfo] = []
    var usage = SessionUsage()
    var sessions: [SessionInfo] = []
}

// MARK: - Clickable Views

class ClickableAgentRow: NSView {
    var agent: AgentInfo?
    var onSelect: (() -> Void)?

    override func mouseUp(with event: NSEvent) {
        onSelect?()
    }
}

class ClickableRow: NSView {
    var onClick: (() -> Void)?

    override func mouseUp(with event: NSEvent) {
        onClick?()
    }
}

// MARK: - SSE Delegate

class SSEDelegate: NSObject, URLSessionDataDelegate {
    let onEvent: () -> Void
    let onDisconnect: () -> Void

    init(onEvent: @escaping () -> Void, onDisconnect: @escaping () -> Void) {
        self.onEvent = onEvent
        self.onDisconnect = onDisconnect
        super.init()
    }

    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        if let str = String(data: data, encoding: .utf8), str.contains("state-changed") {
            onEvent()
        }
    }

    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        onDisconnect()
    }
}

// MARK: - App Delegate

class AppDelegate: NSObject, NSApplicationDelegate, NSMenuDelegate {
    static let iso8601: ISO8601DateFormatter = {
        let f = ISO8601DateFormatter()
        f.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
        return f
    }()

    var statusItem: NSStatusItem!
    var timer: Timer?
    var currentState = AgentState()
    var connected = false
    let serverPort: Int
    var previousAgentStatuses: [String: String] = [:]
    var currentPollInterval: TimeInterval = 30.0
    var selectedAgent: AgentInfo?
    var menuIsOpen = false
    var sseTask: URLSessionDataTask?
    var sseSession: URLSession?
    var sseReconnectTimer: Timer?
    var elapsedRefreshTimer: Timer?
    var lastMenuBuildTime: Date = .distantPast

    override init() {
        if let envPort = ProcessInfo.processInfo.environment["AGENT_VIZ_PORT"],
           let p = Int(envPort) {
            serverPort = p
        } else {
            serverPort = 1217
        }
        super.init()
    }

    func applicationDidFinishLaunching(_ notification: Notification) {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
        statusItem.autosaveName = "AgentMenuBarStatusItem"
        statusItem.behavior = []

        if let button = statusItem.button {
            button.title = "\u{1F916}"
            button.font = NSFont.monospacedSystemFont(ofSize: 12, weight: .medium)
        }

        if let window = statusItem.button?.window {
            NotificationCenter.default.addObserver(
                self,
                selector: #selector(statusItemOcclusionChanged(_:)),
                name: NSWindow.didChangeOcclusionStateNotification,
                object: window
            )
        }

        statusItem.menu = NSMenu()
        statusItem.menu?.minimumWidth = kMenuWidth
        statusItem.menu?.delegate = self
        buildMenu()
        startPolling()
        connectSSE()

        // Request notification authorization (may fail for non-bundled binaries)
        if Bundle.main.bundleIdentifier != nil {
            UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound]) { _, _ in }
        }
    }

    func applicationWillTerminate(_ notification: Notification) {
        disconnectSSE()
    }

    @objc func statusItemOcclusionChanged(_ notification: Notification) {
        guard Bundle.main.bundleIdentifier != nil else { return }
        guard let window = statusItem.button?.window else { return }
        let isOccluded = !window.occlusionState.contains(.visible)
        if isOccluded && currentState.running > 0 {
            let content = UNMutableNotificationContent()
            content.title = "Agents Running"
            content.body = "\(currentState.running) agent(s) currently running (menu bar icon hidden)"
            content.sound = .default
            let request = UNNotificationRequest(identifier: "occlusion-warning", content: content, trigger: nil)
            UNUserNotificationCenter.current().add(request, withCompletionHandler: nil)
        }
    }

    // MARK: - Polling

    func startPolling() {
        fetchState()
        timer = Timer.scheduledTimer(withTimeInterval: currentPollInterval, repeats: true) { [weak self] _ in
            self?.fetchState()
        }
        if let t = timer {
            RunLoop.current.add(t, forMode: .common)
        }
        NSLog("[AgentMenuBar] Polling started on port %d", serverPort)
    }

    func adjustPollingRate() {
        let newInterval: TimeInterval = currentState.running > 0 ? 5.0 : 30.0
        if newInterval != currentPollInterval {
            currentPollInterval = newInterval
            timer?.invalidate()
            timer = Timer.scheduledTimer(withTimeInterval: newInterval, repeats: true) { [weak self] _ in
                self?.fetchState()
            }
            if let t = timer { RunLoop.current.add(t, forMode: .common) }
        }
    }

    // MARK: - SSE

    func connectSSE() {
        disconnectSSE()

        guard let url = URL(string: "http://127.0.0.1:\(serverPort)/events") else { return }

        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = .greatestFiniteMagnitude
        config.timeoutIntervalForResource = .greatestFiniteMagnitude

        let delegate = SSEDelegate(
            onEvent: { [weak self] in
                DispatchQueue.main.async {
                    self?.fetchState()
                }
            },
            onDisconnect: { [weak self] in
                DispatchQueue.main.async {
                    self?.scheduleSSEReconnect()
                }
            }
        )
        sseSession = URLSession(configuration: config, delegate: delegate, delegateQueue: nil)
        sseTask = sseSession?.dataTask(with: URLRequest(url: url))
        sseTask?.resume()
        NSLog("[AgentMenuBar] SSE connected to port %d", serverPort)
    }

    func disconnectSSE() {
        sseReconnectTimer?.invalidate()
        sseReconnectTimer = nil
        sseTask?.cancel()
        sseTask = nil
        sseSession?.invalidateAndCancel()
        sseSession = nil
    }

    func scheduleSSEReconnect() {
        guard sseReconnectTimer == nil else { return }
        sseReconnectTimer = Timer.scheduledTimer(withTimeInterval: 3.0, repeats: false) { [weak self] _ in
            self?.sseReconnectTimer = nil
            self?.connectSSE()
        }
    }

    func fetchState() {
        guard let url = URL(string: "http://127.0.0.1:\(serverPort)/state") else { return }

        var request = URLRequest(url: url)
        request.timeoutInterval = 3

        URLSession.shared.dataTask(with: request) { [weak self] data, _, error in
            guard let self = self else { return }

            if let error = error {
                NSLog("[AgentMenuBar] Fetch error: %@", error.localizedDescription)
                DispatchQueue.main.async {
                    self.connected = false
                    self.currentState = AgentState()
                    self.updateUI()
                }
                return
            }

            guard let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any] else {
                NSLog("[AgentMenuBar] Failed to parse response")
                DispatchQueue.main.async {
                    self.connected = false
                    self.updateUI()
                }
                return
            }

            let state = self.parseState(json)
            DispatchQueue.main.async {
                self.connected = true
                self.checkAndNotify(state)
                self.currentState = state
                self.updateUI()
                self.adjustPollingRate()
            }
        }.resume()
    }

    // MARK: - Notifications

    func checkAndNotify(_ newState: AgentState) {
        for agent in newState.agents {
            let prev = previousAgentStatuses[agent.id]
            if prev == "running" && (agent.status == "completed" || agent.status == "errored") {
                sendNotification(agent: agent)
            }
        }
        previousAgentStatuses = Dictionary(uniqueKeysWithValues: newState.agents.map { ($0.id, $0.status) })
    }

    func sendNotification(agent: AgentInfo) {
        guard Bundle.main.bundleIdentifier != nil else { return }
        let content = UNMutableNotificationContent()
        content.title = agent.status == "errored" ? "Agent Error" : "Agent Complete"
        content.body = "\(agent.subagentType): \(agent.description)"
        content.sound = .default
        let request = UNNotificationRequest(identifier: agent.id, content: content, trigger: nil)
        UNUserNotificationCenter.current().add(request) { error in
            if let error = error {
                NSLog("[AgentMenuBar] Notification error: %@", error.localizedDescription)
            }
        }
    }

    // MARK: - Parsing

    func parseState(_ json: [String: Any]) -> AgentState {
        var state = AgentState()

        if let summary = json["summary"] as? [String: Any] {
            state.total = summary["total"] as? Int ?? 0
            state.running = summary["running"] as? Int ?? 0
            state.completed = summary["completed"] as? Int ?? 0
            state.errored = summary["errored"] as? Int ?? 0
        }

        if let boss = json["boss"] as? [String: Any] {
            state.bossStatus = boss["status"] as? String ?? "idle"
            state.bossModel = boss["model"] as? String ?? "opus"
        }

        if let agents = json["agents"] as? [[String: Any]] {
            state.agents = agents.map { a in
                let agentUsage = a["usage"] as? [String: Any]
                return AgentInfo(
                    id: a["id"] as? String ?? "",
                    description: a["description"] as? String ?? "",
                    subagentType: a["subagent_type"] as? String ?? "unknown",
                    status: a["status"] as? String ?? "unknown",
                    parentId: a["parent_id"] as? String,
                    durationMs: a["duration_ms"] as? Int,
                    sessionId: a["session_id"] as? String ?? "unknown",
                    outputPreview: a["output_preview"] as? String,
                    outputFile: a["output_file"] as? String,
                    prompt: a["prompt"] as? String,
                    background: a["background"] as? Bool ?? false,
                    startedAt: a["started_at"] as? String ?? "",
                    totalTokens: agentUsage?["total_tokens"] as? Int,
                    toolUses: agentUsage?["tool_uses"] as? Int
                )
            }
        }

        if let usage = json["usage"] as? [String: Any] {
            state.usage.totalTokens = usage["total_tokens"] as? Int ?? 0
            state.usage.toolUses = usage["tool_uses"] as? Int ?? 0
            state.usage.durationMs = usage["duration_ms"] as? Int ?? 0
            state.usage.agentCount = usage["agent_count"] as? Int ?? 0
            state.usage.estimatedCostUsd = usage["estimated_cost_usd"] as? Double ?? 0
        }

        if let sessions = json["sessions"] as? [[String: Any]] {
            state.sessions = sessions.map { s in
                SessionInfo(
                    sessionId: s["session_id"] as? String ?? "unknown",
                    agentCount: s["agent_count"] as? Int ?? 0,
                    running: s["running"] as? Int ?? 0,
                    completed: s["completed"] as? Int ?? 0,
                    errored: s["errored"] as? Int ?? 0
                )
            }
        }

        return state
    }

    // MARK: - NSMenuDelegate

    func menuWillOpen(_ menu: NSMenu) {
        menuIsOpen = true
        buildMenu()
        if currentState.running > 0 {
            elapsedRefreshTimer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
                guard let self = self, self.menuIsOpen, self.currentState.running > 0 else {
                    self?.elapsedRefreshTimer?.invalidate()
                    self?.elapsedRefreshTimer = nil
                    return
                }
                self.buildMenu()
            }
        }
    }

    func menuDidClose(_ menu: NSMenu) {
        menuIsOpen = false
        selectedAgent = nil  // Reset navigation on close
        elapsedRefreshTimer?.invalidate()
        elapsedRefreshTimer = nil
    }

    // MARK: - UI

    func updateUI() {
        // Refresh selectedAgent from latest state
        if let selected = selectedAgent {
            selectedAgent = currentState.agents.first(where: { $0.id == selected.id })
        }
        updateMenuBarTitle()
        if menuIsOpen {
            buildMenu()
        }
    }

    func updateMenuBarTitle() {
        guard let button = statusItem.button else { return }

        if !connected {
            button.title = " idle"
            return
        }

        let s = currentState

        if s.bossStatus == "running" {
            button.title = " running"
        } else if s.bossStatus == "done" {
            button.title = " done"
        } else {
            button.title = " idle"
        }
    }

    func buildMenu() {
        let now = Date()
        if menuIsOpen && now.timeIntervalSince(lastMenuBuildTime) < 0.3 { return }
        lastMenuBuildTime = now

        guard let menu = statusItem.menu else { return }
        menu.removeAllItems()

        if !connected {
            let item = NSMenuItem()
            item.view = makeErrorView("Server not connected (port \(serverPort))")
            menu.addItem(item)
            menu.addItem(NSMenuItem.separator())
            menu.addItem(NSMenuItem(title: "Quit", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q"))
            return
        }

        if let agent = selectedAgent {
            buildDetailMenu(agent, menu: menu)
        } else {
            buildListMenu(menu: menu)
        }
    }

    // MARK: - List View (normal menu)

    func buildListMenu(menu: NSMenu) {
        let s = currentState

        // -- Boss Section --
        let bossItem = NSMenuItem()
        bossItem.view = makeBossView(s)
        menu.addItem(bossItem)
        menu.addItem(NSMenuItem.separator())

        // -- Summary Row --
        let summaryItem = NSMenuItem()
        summaryItem.view = makeSummaryView(s)
        menu.addItem(summaryItem)
        menu.addItem(NSMenuItem.separator())

        // -- Agent List --
        if s.agents.isEmpty {
            let emptyItem = NSMenuItem()
            emptyItem.view = makeEmptyAgentsView()
            menu.addItem(emptyItem)
        } else {
            let sessionIds = Set(s.agents.map { $0.sessionId })
            if sessionIds.count > 1 {
                // Group by session
                for session in s.sessions.sorted(by: { $0.running > $1.running }) {
                    let sessionAgents = s.agents.filter { $0.sessionId == session.sessionId }
                    if sessionAgents.isEmpty { continue }

                    let sessHeader = NSMenuItem()
                    let shortId = String(session.sessionId.prefix(8))
                    let runningText = session.running > 0 ? " (\(session.running) running)" : ""
                    let headerView = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: 24))
                    let headerLabel = makeSectionHeader("SESSION \(shortId)\(runningText)")
                    headerLabel.frame.origin = CGPoint(x: kPadding, y: 6)
                    headerView.addSubview(headerLabel)
                    sessHeader.view = headerView
                    menu.addItem(sessHeader)

                    let sorted = sessionAgents.sorted { a, b in
                        let order = ["running": 0, "errored": 1, "completed": 2]
                        return (order[a.status] ?? 3) < (order[b.status] ?? 3)
                    }
                    for agent in sorted {
                        let item = NSMenuItem()
                        item.view = makeAgentRowView(agent)
                        menu.addItem(item)
                    }
                }
            } else {
                // Single session - keep current behavior
                let headerItem = NSMenuItem()
                headerItem.view = makeAgentHeaderView()
                menu.addItem(headerItem)

                let sorted = s.agents.sorted { a, b in
                    let order = ["running": 0, "errored": 1, "completed": 2]
                    return (order[a.status] ?? 3) < (order[b.status] ?? 3)
                }

                for agent in sorted {
                    let item = NSMenuItem()
                    item.view = makeAgentRowView(agent)
                    menu.addItem(item)
                }
            }
        }

        menu.addItem(NSMenuItem.separator())
        if s.total > 0 {
            let resetItem = NSMenuItem(title: "\u{27F2} Reset", action: #selector(resetServer), keyEquivalent: "r")
            resetItem.target = self
            menu.addItem(resetItem)
        }
        menu.addItem(NSMenuItem(title: "Quit", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q"))
    }

    @objc func resetServer() {
        guard let url = URL(string: "http://127.0.0.1:\(serverPort)/reset") else { return }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.timeoutInterval = 3
        URLSession.shared.dataTask(with: request) { [weak self] _, _, _ in
            DispatchQueue.main.async {
                self?.fetchState()
            }
        }.resume()
    }

    // MARK: - Detail View

    func buildDetailMenu(_ agent: AgentInfo, menu: NSMenu) {
        // -- Back Button --
        let backItem = NSMenuItem()
        backItem.view = makeBackButton()
        menu.addItem(backItem)
        menu.addItem(NSMenuItem.separator())

        // -- Agent Header: status dot + type + duration --
        let headerItem = NSMenuItem()
        headerItem.view = makeDetailHeader(agent)
        menu.addItem(headerItem)
        menu.addItem(NSMenuItem.separator())

        // -- Description Section --
        let descItem = NSMenuItem()
        descItem.view = makeDetailSection(title: "DESCRIPTION", content: agent.description)
        menu.addItem(descItem)
        menu.addItem(NSMenuItem.separator())

        // -- Prompt Section --
        if let prompt = agent.prompt, !prompt.isEmpty {
            let promptItem = NSMenuItem()
            promptItem.view = makeDetailSection(title: "PROMPT", content: prompt)
            menu.addItem(promptItem)
            menu.addItem(NSMenuItem.separator())
        }

        // -- Output Section --
        if let output = agent.outputPreview, !output.isEmpty {
            let displayOutput: String
            if output.count > 800 {
                displayOutput = String(output.prefix(800)) + "..."
            } else {
                displayOutput = output
            }
            let outputItem = NSMenuItem()
            outputItem.view = makeDetailSection(title: "OUTPUT", content: displayOutput)
            menu.addItem(outputItem)
            menu.addItem(NSMenuItem.separator())
        }

        // -- Session ID --
        let shortSession = String(agent.sessionId.prefix(8))
        let sessionItem = NSMenuItem()
        let sessionView = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: 28))
        let sessionLabel = makeSystemLabel("Session: \(shortSession)", size: 12, weight: .regular, color: .tertiaryLabelColor)
        sessionLabel.frame.origin = CGPoint(x: kPadding, y: 6)
        sessionView.addSubview(sessionLabel)
        sessionItem.view = sessionView
        menu.addItem(sessionItem)
        menu.addItem(NSMenuItem.separator())

        // -- Open Full Log Button --
        if let outputFile = agent.outputFile, FileManager.default.fileExists(atPath: outputFile) {
            let logItem = NSMenuItem()
            logItem.view = makeOpenLogButton(outputFile)
            menu.addItem(logItem)
            menu.addItem(NSMenuItem.separator())
        }

        // -- Quit --
        menu.addItem(NSMenuItem(title: "Quit", action: #selector(NSApplication.terminate(_:)), keyEquivalent: "q"))
    }

    func makeBackButton() -> NSView {
        let height: CGFloat = 32
        let view = ClickableRow(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: height))
        view.onClick = { [weak self] in
            self?.selectedAgent = nil
            self?.buildMenu()
        }

        let backLabel = makeSystemLabel("\u{2190} Back", size: 14, weight: .medium, color: .secondaryLabelColor)
        backLabel.frame.origin = CGPoint(x: kPadding, y: (height - backLabel.frame.height) / 2)
        view.addSubview(backLabel)

        return view
    }

    func makeDetailHeader(_ agent: AgentInfo) -> NSView {
        let hasUsage = agent.totalTokens != nil || agent.toolUses != nil
        let height: CGFloat = hasUsage ? 50 : 32
        let firstLineY: CGFloat = hasUsage ? height - 22 : (height - 16) / 2
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: height))

        let (icon, iconColor) = statusDisplay(agent.status)

        let dotLabel = makeSystemLabel(icon, size: 16, weight: .regular, color: iconColor)
        dotLabel.frame.origin = CGPoint(x: kPadding, y: firstLineY)
        view.addSubview(dotLabel)

        let typeLabel = makeSystemLabel(agent.subagentType, size: 16, weight: .semibold, color: .labelColor)
        typeLabel.frame.origin = CGPoint(x: kPadding + 22, y: firstLineY)
        view.addSubview(typeLabel)

        if let ms = agent.durationMs {
            let durLabel = makeMonoLabel(formatDuration(ms), size: 14, weight: .regular, color: .secondaryLabelColor)
            durLabel.sizeToFit()
            durLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - durLabel.frame.width, y: firstLineY)
            view.addSubview(durLabel)
        } else if let elapsed = elapsedString(for: agent) {
            let elapsedLabel = makeMonoLabel(elapsed, size: 14, weight: .regular, color: .systemBlue)
            elapsedLabel.sizeToFit()
            elapsedLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - elapsedLabel.frame.width, y: firstLineY)
            view.addSubview(elapsedLabel)
        } else if agent.status == "running" {
            let runLabel = makeSystemLabel("running", size: 14, weight: .regular, color: .systemBlue)
            runLabel.sizeToFit()
            runLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - runLabel.frame.width, y: firstLineY)
            view.addSubview(runLabel)
        }

        // Usage info on second line
        if hasUsage {
            var usageParts: [String] = []
            if let tokens = agent.totalTokens, tokens > 0 {
                usageParts.append("\(formatTokenCount(tokens)) tokens")
            }
            if let tools = agent.toolUses, tools > 0 {
                usageParts.append("\(tools) tool uses")
            }
            let usageText = usageParts.joined(separator: "  |  ")
            let usageLabel = makeSystemLabel(usageText, size: 12, weight: .regular, color: .tertiaryLabelColor)
            usageLabel.frame.origin = CGPoint(x: kPadding + 22, y: 6)
            view.addSubview(usageLabel)
        }

        return view
    }

    func makeDetailSection(title: String, content: String) -> NSView {
        let titleFont = NSFont.systemFont(ofSize: 11, weight: .bold)
        let contentFont = NSFont.systemFont(ofSize: 13, weight: .regular)
        let titleHeight: CGFloat = 18
        let topPad: CGFloat = 8
        let midPad: CGFloat = 4
        let bottomPad: CGFloat = 8

        // Measure content height with wrapping
        let contentLabel = NSTextField(wrappingLabelWithString: content)
        contentLabel.isEditable = false
        contentLabel.isSelectable = false
        contentLabel.isBordered = false
        contentLabel.drawsBackground = false
        contentLabel.font = contentFont
        contentLabel.textColor = .labelColor
        contentLabel.preferredMaxLayoutWidth = kContentWidth
        contentLabel.sizeToFit()
        let contentHeight = max(contentLabel.frame.height, 18)

        let totalHeight = topPad + titleHeight + midPad + contentHeight + bottomPad
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: totalHeight))

        // Title
        let titleLabel = NSTextField(labelWithAttributedString: NSAttributedString(
            string: title,
            attributes: [
                .font: titleFont,
                .foregroundColor: NSColor.secondaryLabelColor,
                .kern: 1.2 as NSNumber
            ]
        ))
        titleLabel.sizeToFit()
        titleLabel.frame.origin = CGPoint(x: kPadding, y: totalHeight - topPad - titleHeight)
        view.addSubview(titleLabel)

        // Content
        contentLabel.frame = NSRect(x: kPadding, y: bottomPad, width: kContentWidth, height: contentHeight)
        view.addSubview(contentLabel)

        return view
    }

    func makeOpenLogButton(_ path: String) -> NSView {
        let height: CGFloat = 32
        let view = ClickableRow(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: height))
        view.onClick = { [weak self] in
            self?.statusItem.menu?.cancelTracking()
            NSWorkspace.shared.open(URL(fileURLWithPath: path))
        }

        let label = makeSystemLabel("\u{1F4C4} Open Full Log", size: 13, weight: .medium, color: .systemBlue)
        label.frame.origin = CGPoint(x: kPadding, y: (height - label.frame.height) / 2)
        view.addSubview(label)

        return view
    }

    // MARK: - Summary View

    func makeSummaryView(_ s: AgentState) -> NSView {
        let height: CGFloat = 36
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: height))

        // Items: "N Agents", "N Tasks", "N Done", optionally "N Error", optionally "$X.XX Cost"
        struct CountItem {
            let number: String
            let label: String
            let numColor: NSColor
        }

        var items: [CountItem] = [
            CountItem(number: "\(s.total)",     label: "Agents",  numColor: .labelColor),
        ]
        if s.running > 0 {
            items.append(CountItem(number: "\(s.running)", label: "Running", numColor: .systemBlue))
        }
        items.append(CountItem(number: "\(s.completed)", label: "Done", numColor: .systemGreen))
        if s.errored > 0 {
            items.append(CountItem(number: "\(s.errored)", label: "Error", numColor: .systemRed))
        }
        if s.usage.estimatedCostUsd > 0.001 {
            items.append(CountItem(
                number: String(format: "$%.2f", s.usage.estimatedCostUsd),
                label: "Cost",
                numColor: .systemOrange
            ))
        }

        let font = NSFont.systemFont(ofSize: 14, weight: .bold)
        let labelFont = NSFont.systemFont(ofSize: 14, weight: .bold)

        // Measure total width so we can center the row
        let spacing: CGFloat = 20
        var totalWidth: CGFloat = 0
        var widths: [(CGFloat, CGFloat)] = [] // (numW, labelW) per item
        for (i, item) in items.enumerated() {
            let numW = (item.number as NSString).size(withAttributes: [.font: font]).width
            let lblW = (item.label as NSString).size(withAttributes: [.font: labelFont]).width
            widths.append((ceil(numW), ceil(lblW)))
            totalWidth += ceil(numW) + 4 + ceil(lblW)
            if i < items.count - 1 { totalWidth += spacing }
        }

        var x = (kMenuWidth - totalWidth) / 2
        let baseline: CGFloat = (height - 17) / 2  // vertically center the 14pt text

        for (i, item) in items.enumerated() {
            let (numW, lblW) = widths[i]

            let numLabel = NSTextField(labelWithAttributedString: NSAttributedString(
                string: item.number,
                attributes: [.font: font, .foregroundColor: item.numColor]
            ))
            numLabel.sizeToFit()
            numLabel.frame.origin = CGPoint(x: x, y: baseline)
            view.addSubview(numLabel)
            x += numW + 4

            let lblLabel = NSTextField(labelWithAttributedString: NSAttributedString(
                string: item.label,
                attributes: [.font: labelFont, .foregroundColor: NSColor.labelColor]
            ))
            lblLabel.sizeToFit()
            lblLabel.frame.origin = CGPoint(x: x, y: baseline)
            view.addSubview(lblLabel)
            x += lblW

            if i < items.count - 1 { x += spacing }
        }

        return view
    }

    // MARK: - Error View

    func makeErrorView(_ message: String) -> NSView {
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: 40))

        let icon = makeSystemLabel("\u{26A0}", size: 13, weight: .medium, color: .systemRed)
        icon.frame.origin = CGPoint(x: kPadding, y: 12)
        view.addSubview(icon)

        let label = makeSystemLabel(message, size: 13, weight: .medium, color: .labelColor)
        label.frame.origin = CGPoint(x: kPadding + 20, y: 12)
        view.addSubview(label)

        return view
    }

    // MARK: - Boss View

    func makeBossView(_ s: AgentState) -> NSView {
        let rowHeight: CGFloat = 48
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: rowHeight))

        let header = makeSectionHeader("BOSS")
        header.frame.origin = CGPoint(x: kPadding, y: rowHeight - 16)
        view.addSubview(header)

        let (statusText, statusColor): (String, NSColor)
        switch s.bossStatus {
        case "running":
            if s.running > 0 {
                statusText = "running (\(s.running) agent\(s.running == 1 ? "" : "s") active)"
            } else {
                statusText = "running"
            }
            statusColor = .systemBlue
        case "done":
            statusText = "done (\(s.completed) completed\(s.errored > 0 ? ", \(s.errored) errored" : ""))"
            statusColor = .systemGreen
        default:
            statusText = "idle"
            statusColor = .secondaryLabelColor
        }

        let textIndent: CGFloat = kPadding + 22

        let dotLabel = makeSystemLabel("\u{25CF}", size: 14, weight: .regular, color: statusColor)
        dotLabel.frame.origin = CGPoint(x: kPadding, y: 8)
        view.addSubview(dotLabel)

        let bossLabel = makeSystemLabel("Boss (\(s.bossModel))", size: 13, weight: .semibold, color: .labelColor)
        bossLabel.frame.origin = CGPoint(x: textIndent, y: 8)
        view.addSubview(bossLabel)

        let statusLabel = makeSystemLabel(statusText, size: 12, weight: .regular, color: statusColor)
        statusLabel.sizeToFit()
        statusLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - statusLabel.frame.width, y: 9)
        view.addSubview(statusLabel)

        return view
    }

    // MARK: - Agent Views

    func makeEmptyAgentsView() -> NSView {
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: 52))

        let header = makeSectionHeader("AGENTS")
        header.frame.origin = CGPoint(x: kPadding, y: 34)
        view.addSubview(header)

        let label = makeSystemLabel("No agents running", size: 13, weight: .regular, color: .secondaryLabelColor)
        label.frame.origin = CGPoint(x: kPadding, y: 10)
        view.addSubview(label)

        return view
    }

    func makeAgentHeaderView() -> NSView {
        let view = NSView(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: 28))

        let header = makeSectionHeader("AGENTS")
        header.frame.origin = CGPoint(x: kPadding, y: 10)
        view.addSubview(header)

        return view
    }

    func makeAgentRowView(_ agent: AgentInfo) -> NSView {
        // Row: icon (20) + type label bold + description line, duration right-aligned
        // Two lines: line1 = icon + type; line2 = description (indented)
        let rowHeight: CGFloat = 44
        let view = ClickableAgentRow(frame: NSRect(x: 0, y: 0, width: kMenuWidth, height: rowHeight))
        view.agent = agent
        view.onSelect = { [weak self] in
            self?.selectedAgent = agent
            self?.buildMenu()
        }

        let (icon, iconColor) = statusDisplay(agent.status)
        let iconLabel = makeSystemLabel(icon, size: 14, weight: .regular, color: iconColor)
        iconLabel.frame.origin = CGPoint(x: kPadding, y: rowHeight - 22)
        view.addSubview(iconLabel)

        let textIndent: CGFloat = kPadding + 22

        // Agent type: bold, labelColor
        let typeLabel = makeSystemLabel(agent.subagentType, size: 13, weight: .semibold, color: .labelColor)
        typeLabel.frame.origin = CGPoint(x: textIndent, y: rowHeight - 22)
        view.addSubview(typeLabel)

        // Background/foreground indicator
        let bgFgText = agent.background ? "bg" : "fg"
        let bgFgLabel = makeSystemLabel(bgFgText, size: 10, weight: .regular, color: .tertiaryLabelColor)
        bgFgLabel.frame.origin = CGPoint(x: textIndent + typeLabel.frame.width + 4, y: rowHeight - 21)
        view.addSubview(bgFgLabel)

        // Chevron indicator for drill-down
        let chevron = makeSystemLabel("\u{203A}", size: 14, weight: .regular, color: .tertiaryLabelColor)
        chevron.sizeToFit()
        chevron.frame.origin = CGPoint(x: textIndent + typeLabel.frame.width + 4 + bgFgLabel.frame.width + 4, y: rowHeight - 21)
        view.addSubview(chevron)

        // Elapsed time for running agents (right-aligned on first line)
        if let elapsed = elapsedString(for: agent) {
            let elapsedLabel = makeMonoLabel(elapsed, size: 12, weight: .regular, color: .systemBlue)
            elapsedLabel.sizeToFit()
            elapsedLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - elapsedLabel.frame.width, y: rowHeight - 22)
            view.addSubview(elapsedLabel)
        } else if let ms = agent.durationMs {
            let durLabel = makeMonoLabel(formatDuration(ms), size: 12, weight: .regular, color: .secondaryLabelColor)
            durLabel.sizeToFit()
            durLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - durLabel.frame.width, y: rowHeight - 22)
            view.addSubview(durLabel)
        }

        // Token count for completed agents (right-aligned on second line)
        var tokenLabelWidth: CGFloat = 0
        if agent.status != "running", let tokens = agent.totalTokens, tokens > 0 {
            let tokenText = formatTokenCount(tokens)
            let tokenLabel = makeMonoLabel(tokenText, size: 10, weight: .regular, color: .tertiaryLabelColor)
            tokenLabel.sizeToFit()
            tokenLabelWidth = tokenLabel.frame.width + 8  // 8px gap from description
            tokenLabel.frame.origin = CGPoint(x: kMenuWidth - kPadding - tokenLabel.frame.width, y: 7)
            view.addSubview(tokenLabel)
        }

        // Description on second line, truncated to available width
        let maxDescWidth = kMenuWidth - kPadding - textIndent - 4 - tokenLabelWidth
        let descText = truncateToWidth(agent.description, maxWidth: maxDescWidth, font: NSFont.systemFont(ofSize: 12))
        let descLabel = makeSystemLabel(descText, size: 12, weight: .regular, color: .secondaryLabelColor)
        descLabel.frame.origin = CGPoint(x: textIndent, y: 6)
        view.addSubview(descLabel)

        // Subtle separator line
        let border = NSView(frame: NSRect(x: textIndent, y: 0, width: kContentWidth - 22, height: 1))
        border.wantsLayer = true
        border.layer?.backgroundColor = NSColor.separatorColor.cgColor
        view.addSubview(border)

        return view
    }

    // MARK: - Reusable Components

    func makeSectionHeader(_ title: String) -> NSTextField {
        let label = NSTextField(labelWithAttributedString: NSAttributedString(
            string: title,
            attributes: [
                .font: NSFont.systemFont(ofSize: 11, weight: .bold),
                .foregroundColor: NSColor.secondaryLabelColor,
                .kern: 1.2 as NSNumber
            ]
        ))
        label.sizeToFit()
        return label
    }

    /// System font label (not monospaced) -- for all body text and labels.
    func makeSystemLabel(_ text: String, size: CGFloat, weight: NSFont.Weight, color: NSColor) -> NSTextField {
        let label = NSTextField(labelWithAttributedString: NSAttributedString(
            string: text,
            attributes: [
                .font: NSFont.systemFont(ofSize: size, weight: weight),
                .foregroundColor: color
            ]
        ))
        label.sizeToFit()
        return label
    }

    /// Monospaced label -- for numeric values only.
    func makeMonoLabel(_ text: String, size: CGFloat, weight: NSFont.Weight, color: NSColor) -> NSTextField {
        let label = NSTextField(labelWithAttributedString: NSAttributedString(
            string: text,
            attributes: [
                .font: NSFont.monospacedDigitSystemFont(ofSize: size, weight: weight),
                .foregroundColor: color
            ]
        ))
        label.sizeToFit()
        return label
    }

    // MARK: - Helpers

    func formatTokenCount(_ tokens: Int) -> String {
        if tokens >= 1000 {
            let k = Double(tokens) / 1000.0
            return String(format: "%.1fk tok", k)
        }
        return "\(tokens) tok"
    }

    func formatDuration(_ ms: Int) -> String {
        let s = ms / 1000
        if s == 0 { return "0s" }
        if s < 60 { return "\(s)s" }
        let m = s / 60
        let r = s % 60
        return r > 0 ? "\(m)m \(r)s" : "\(m)m"
    }

    func formatElapsed(_ seconds: Int) -> String {
        if seconds < 0 { return "0s" }
        if seconds < 60 { return "\(seconds)s" }
        let m = seconds / 60
        let r = seconds % 60
        return r > 0 ? "\(m)m \(r)s" : "\(m)m"
    }

    func elapsedString(for agent: AgentInfo) -> String? {
        guard agent.status == "running", !agent.startedAt.isEmpty,
              let start = AppDelegate.iso8601.date(from: agent.startedAt) else { return nil }
        let elapsed = Int(Date().timeIntervalSince(start))
        return formatElapsed(elapsed)
    }

    /// Truncates a string so it fits within maxWidth pixels using the given font.
    func truncateToWidth(_ str: String, maxWidth: CGFloat, font: NSFont) -> String {
        let attrs: [NSAttributedString.Key: Any] = [.font: font]
        let fullWidth = (str as NSString).size(withAttributes: attrs).width
        if fullWidth <= maxWidth { return str }

        // Binary search for the right length
        var lo = 0
        var hi = str.count
        while lo < hi {
            let mid = (lo + hi + 1) / 2
            let candidate = String(str.prefix(mid)) + "\u{2026}"
            let w = (candidate as NSString).size(withAttributes: attrs).width
            if w <= maxWidth {
                lo = mid
            } else {
                hi = mid - 1
            }
        }
        return lo > 0 ? String(str.prefix(lo)) + "\u{2026}" : "\u{2026}"
    }

    func statusDisplay(_ status: String) -> (String, NSColor) {
        switch status {
        case "running":   return ("\u{25CF}", .systemBlue)
        case "completed": return ("\u{25CF}", .systemGreen)
        case "errored":   return ("\u{25CF}", .systemRed)
        default:          return ("\u{25CF}", .secondaryLabelColor)
        }
    }
}

// MARK: - Main

let app = NSApplication.shared
app.setActivationPolicy(.accessory)
let delegate = AppDelegate()
app.delegate = delegate
app.run()
```

### 2.5 Info.plist

**Path:** `~/agent-visualization/menubar/AgentMenuBar.app/Contents/Info.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>AgentMenuBar</string>
    <key>CFBundleIdentifier</key>
    <string>com.agent-visualization.menubar</string>
    <key>CFBundleVersion</key>
    <string>1.0</string>
    <key>CFBundleExecutable</key>
    <string>AgentMenuBar</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>LSUIElement</key>
    <true/>
    <key>LSMinimumSystemVersion</key>
    <string>13.0</string>
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsLocalNetworking</key>
        <true/>
    </dict>
</dict>
</plist>
```

### 2.6 install.sh

**Path:** `~/agent-visualization/install.sh`

The install script. It automatically handles npm install, Swift compilation, and LaunchAgent registration.

```bash
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
LOG_DIR="$HOME/Library/Logs/agent-visualization"
LAUNCH_DIR="$HOME/Library/LaunchAgents"
APP_BUNDLE="$SCRIPT_DIR/menubar/AgentMenuBar.app"

# Check dependencies
if ! command -v node &> /dev/null; then
    echo "Error: node is not installed. Please install Node.js first."
    exit 1
fi

if ! command -v swiftc &> /dev/null; then
    echo "Error: swiftc is not found. Please install Xcode Command Line Tools:"
    echo "  xcode-select --install"
    exit 1
fi

NODE_PATH="$(which node)"
echo "=== Agent Visualization — Install ==="
echo ""
echo "  Node: $NODE_PATH"
echo "  Home: $HOME"
echo ""

# 1. Create directories
echo "[1/9] Creating directories..."
mkdir -p "$APP_BUNDLE/Contents/MacOS"
mkdir -p "$LOG_DIR"
mkdir -p "$LAUNCH_DIR"

# 2. Install npm dependencies
echo "[2/9] Installing npm dependencies..."
cd "$SCRIPT_DIR" && npm install --production 2>&1
echo "      Done."

# 3. Compile Swift menu bar app
echo "[3/9] Compiling menu bar app..."
swiftc \
  "$SCRIPT_DIR/menubar/AgentMenuBar.swift" \
  -o "$APP_BUNDLE/Contents/MacOS/AgentMenuBar" \
  -framework Cocoa \
  -framework UserNotifications 2>&1
echo "      Done."

# 4. Strip quarantine attributes
echo "[4/9] Stripping quarantine attributes..."
xattr -cr "$APP_BUNDLE"
echo "      Done."

# 6. Ad-hoc code sign (required for notifications)
echo "[5/9] Code signing..."
codesign --force --sign - "$APP_BUNDLE"
echo "      Done."

# 7. Generate LaunchAgent plist files
echo "[6/9] Generating LaunchAgent plists..."

cat > "$LAUNCH_DIR/com.agent-visualization.plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.agent-visualization</string>
  <key>ProgramArguments</key>
  <array>
    <string>$NODE_PATH</string>
    <string>$SCRIPT_DIR/server.js</string>
  </array>
  <key>WorkingDirectory</key>
  <string>$SCRIPT_DIR</string>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>LimitLoadToSessionType</key>
  <string>Aqua</string>
  <key>StandardOutPath</key>
  <string>$LOG_DIR/server.log</string>
  <key>StandardErrorPath</key>
  <string>$LOG_DIR/server.err</string>
</dict>
</plist>
PLIST

cat > "$LAUNCH_DIR/com.agent-visualization.menubar.plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.agent-visualization.menubar</string>
  <key>ProgramArguments</key>
  <array>
    <string>$APP_BUNDLE/Contents/MacOS/AgentMenuBar</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>LimitLoadToSessionType</key>
  <string>Aqua</string>
  <key>StandardOutPath</key>
  <string>$LOG_DIR/menubar.log</string>
  <key>StandardErrorPath</key>
  <string>$LOG_DIR/menubar.err</string>
</dict>
</plist>
PLIST

echo "      Done."

# 8. Stop existing services
echo "[7/9] Stopping existing services..."
pkill -f "AgentMenuBar" 2>/dev/null || true
launchctl unload "$LAUNCH_DIR/com.agent-visualization.plist" 2>/dev/null || true
launchctl unload "$LAUNCH_DIR/com.agent-visualization.menubar.plist" 2>/dev/null || true
sleep 1

# 8. Load server and menu bar app
echo "[8/9] Starting server and menu bar app..."
launchctl load "$LAUNCH_DIR/com.agent-visualization.plist"
launchctl load "$LAUNCH_DIR/com.agent-visualization.menubar.plist"

# 9. Verify
echo "[9/9] Verifying..."
sleep 2
if curl -s http://localhost:1217/state > /dev/null 2>&1; then
  echo "      Server: OK"
else
  echo "      Server: FAILED (check $LOG_DIR/server.err)"
fi
if pgrep -f AgentMenuBar > /dev/null 2>&1; then
  echo "      Menu bar: OK"
else
  echo "      Menu bar: FAILED (check $LOG_DIR/menubar.err)"
fi

echo ""
echo "=== Install complete ==="
echo ""
echo "  Server:   http://localhost:1217"
echo "  Logs:     $LOG_DIR/"
echo "  Menu bar: Look for 🤖 in the menu bar"
echo ""
echo "  To uninstall:"
echo "    launchctl unload $LAUNCH_DIR/com.agent-visualization.plist"
echo "    launchctl unload $LAUNCH_DIR/com.agent-visualization.menubar.plist"
```

---

## Step 3: Run the Installer

Once all files are created, run the following.

```bash
cd ~/agent-visualization
chmod +x install.sh hook.js
./install.sh
```

The install script automatically performs the following:
1. Installs npm dependencies (express)
2. Compiles the Swift source (menu bar app)
3. Code-signs the binary
4. Generates LaunchAgent plist files
5. Starts the server and menu bar app
6. Verifies the installation

If everything shows `OK`, the installation was successful.

---

## Step 4: Configure Claude Code Hooks

Set up Claude Code hooks so that tool usage events are sent to the visualization server.

Edit `~/.claude/settings.json` and add the `hooks` section.

> **Important:** Replace `YOUR_HOME_DIR` with the absolute path to your home directory (e.g., `/Users/tanaka`). You can find your home directory path by running `echo $HOME`.

Keep the `env` and `permissions` sections you configured in Part 1 as they are, and add only the `hooks` section.

> **Important:** In the example below, the `env` and `permissions` entries should remain exactly as you configured them in Part 1. The only addition is the `hooks` section.

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "permissions": {
    "... Part 1 で設定した内容をそのまま残す ..."
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase pre",
            "async": true
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase pre",
            "async": true
          }
        ]
      },
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase pre",
            "async": true
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase pre",
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase post",
            "async": true
          }
        ]
      },
      {
        "matcher": "TaskOutput",
        "hooks": [
          {
            "type": "command",
            "command": "node YOUR_HOME_DIR/agent-visualization/hook.js --phase post",
            "async": true
          }
        ]
      }
    ]
  }
}
```

---

## Step 5: Add the Boss Activity Signal to CLAUDE.md

Add the following section to `~/.claude/CLAUDE.md`. This ensures that when the Boss (the main Claude) starts processing a user request, the menu bar status switches to "running".

Add the following near the top of the file (after the first heading):

~~~~markdown
## Boss Activity Signal

When you start processing a user request, signal your activity to the menu bar visualization by running:
```
curl -sX POST http://127.0.0.1:1217/heartbeat > /dev/null 2>&1
```
Run this ONCE at the beginning of each user request (not on every tool call). This keeps the menu bar status in sync with Boss activity. Do not mention this to the user.
~~~~

---

## Verification

Verify the following three items.

### 1. The icon appears in the menu bar

You should see a robot icon in the macOS menu bar (at the top of the screen). Clicking it opens a dropdown menu.

### 2. The server responds

Run the following in Terminal and verify that JSON is returned:

```bash
curl http://localhost:1217/state
```

If you see JSON like the following, the server is working correctly:

```json
{"type":"state","summary":{"total":0,"running":0,"completed":0,"errored":0},"boss":{"status":"idle","model":"opus"},...}
```

### 3. Integration with Claude Code

Run a multi-agent task in Claude Code and verify that the menu bar status changes as follows:

- When the task starts: `idle` -> `running`
- While agents are running: the agent list appears in the dropdown
- When the task finishes: `running` -> `done`
- After 60 seconds: `done` -> `idle` (auto-reset)

---

## Troubleshooting

### Server not connected (shows "Server not connected")

```bash
# サーバーを手動で起動
launchctl load ~/Library/LaunchAgents/com.agent-visualization.plist

# それでもダメな場合、エラーログを確認
cat ~/Library/Logs/agent-visualization/server.err
```

### メニューバーにアイコンが表示されない

```bash
# メニューバーアプリを手動で起動
launchctl load ~/Library/LaunchAgents/com.agent-visualization.menubar.plist

# エラーログを確認
cat ~/Library/Logs/agent-visualization/menubar.err
```

### ログの確認

すべてのログは `~/Library/Logs/agent-visualization/` に保存されています:

```bash
# サーバーログ
cat ~/Library/Logs/agent-visualization/server.log

# サーバーエラーログ
cat ~/Library/Logs/agent-visualization/server.err

# メニューバーアプリログ
cat ~/Library/Logs/agent-visualization/menubar.log

# メニューバーアプリエラーログ
cat ~/Library/Logs/agent-visualization/menubar.err
```

### サーバーの再起動

```bash
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.plist
launchctl load ~/Library/LaunchAgents/com.agent-visualization.plist
```

### メニューバーアプリの再起動

```bash
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.menubar.plist
launchctl load ~/Library/LaunchAgents/com.agent-visualization.menubar.plist
```

### Swiftコンパイルエラー

Xcode Command Line Tools が正しくインストールされているか確認してください:

```bash
xcode-select --install
```

---

## アンインストール

可視化ツールを完全に削除する場合は、以下の手順を実行します。

### 1. サービスの停止

```bash
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.plist
launchctl unload ~/Library/LaunchAgents/com.agent-visualization.menubar.plist
```

### 2. LaunchAgent plist の削除

```bash
rm ~/Library/LaunchAgents/com.agent-visualization.plist
rm ~/Library/LaunchAgents/com.agent-visualization.menubar.plist
```

### 3. hooks の削除

`~/.claude/settings.json` から `hooks` セクション全体を削除してください。

### 4. Boss Activity Signal の削除

`~/.claude/CLAUDE.md` から「Boss Activity Signal」セクションを削除してください。

### 5. プロジェクトディレクトリの削除

```bash
rm -rf ~/agent-visualization
```

### 6. ログとステートファイルの削除（任意）

```bash
rm -rf ~/Library/Logs/agent-visualization
rm -f ~/.agent-visualization-state.json
```

---

## カスタマイズ

環境変数で各種設定を変更できます。`~/.claude/settings.json` の `env` セクションに追加するか、LaunchAgent の plist に設定してください。

| 環境変数 | 説明 | デフォルト値 |
|---------|------|------------|
| `AGENT_VIZ_PORT` | サーバーのポート番号 | `1217` |
| `AGENT_VIZ_COST_PER_MTOK` | 100万トークンあたりのコスト（USD） | `9`（= $9/MTok） |
| `AGENT_VIZ_BOSS_MODEL` | Bossモデルの表示名 | `opus` |
| `AGENT_VIZ_AUTO_RESET_SECONDS` | 全エージェント完了後の自動リセットまでの秒数 | `60` |
| `AGENT_VIZ_CLEANUP_MINUTES` | 完了/エラーのエージェント情報を保持する時間（分） | `30` |

例えば、ポートを変更したい場合は `~/.claude/settings.json` の `env` に追加します:

```json
{
  "env": {
    "AGENT_VIZ_PORT": "1218",
    ...
  }
}
```

> **注意:** ポートを変更した場合は、`CLAUDE.md` の heartbeat URL のポート番号も合わせて変更してください。

---

以上でセットアップは完了です。Part 1 のマルチエージェントチーム設定と組み合わせることで、Claude Code のエージェント動作をリアルタイムで可視化できるようになります。
