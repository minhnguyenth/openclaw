# Plan: Build a Simplified Agent/Subagent System

## Project Overview

Build a standalone TypeScript project called **"mini-agents"** that replicates the core agent/subagent architecture from OpenClaw in a simplified, learnable form. It will use the same tech stack (Node.js, TypeScript ESM, Anthropic SDK) but strip away the multi-channel/plugin complexity so you can focus on the agent coordination patterns.

## Tech Stack

| Technology | Why | OpenClaw equivalent |
|------------|-----|---------------------|
| Node.js 22+ | Runtime | Same |
| TypeScript (ESM, strict) | Language | Same |
| `@anthropic-ai/sdk` | LLM API calls | `@mariozechner/pi-ai` (streamSimple) |
| Vitest | Testing | Same |
| File-based JSON | Session persistence | Same (SessionManager → .jsonl files) |
| Express | HTTP gateway (optional) | Same (gateway server) |

## Architecture Diagram

```
mini-agents/
├── src/
│   ├── index.ts                  # CLI entry point
│   ├── config/
│   │   ├── types.ts              # AgentConfig, AgentsConfig types
│   │   └── loader.ts             # Load agents.json config file
│   ├── agents/
│   │   ├── agent-scope.ts        # Resolve which agent handles a request
│   │   ├── system-prompt.ts      # Build system prompts (full vs minimal)
│   │   └── tools.ts              # Tool definitions (web_search, spawn_subagent, etc.)
│   ├── runner/
│   │   ├── runner.ts             # Main agent execution loop (the "turn")
│   │   ├── stream.ts             # Anthropic streaming wrapper
│   │   └── tool-executor.ts      # Execute tool calls and return results
│   ├── sessions/
│   │   ├── session-store.ts      # File-based session persistence
│   │   └── session-key.ts        # Session key generation/parsing
│   ├── lanes/
│   │   └── command-queue.ts      # Lane-based async task queue
│   ├── subagents/
│   │   ├── spawn.ts              # Spawn subagent runs
│   │   ├── registry.ts           # Track active/completed subagent runs
│   │   ├── depth.ts              # Depth tracking (prevent infinite nesting)
│   │   └── announce.ts           # Deliver subagent results to parent
│   └── gateway/
│       └── server.ts             # HTTP server for agent RPC (optional Phase 4)
├── agents.json                   # Agent configuration file
├── sessions/                     # Session transcript storage (created at runtime)
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

## Implementation Phases

---

### Phase 1: Foundation — Config, Sessions, and a Single Agent

**Goal:** Get one agent running that can have a multi-turn conversation with tool use.

#### Step 1.1: Project scaffold
- Initialize npm project with TypeScript ESM (`"type": "module"`)
- Install deps: `@anthropic-ai/sdk`, `commander` (CLI), `chalk` (colors)
- Configure `tsconfig.json` (strict, NodeNext, ES2023)
- Add `vitest` for testing

#### Step 1.2: Agent config types (`src/config/types.ts`)
Create simplified versions of OpenClaw's types:
```typescript
type AgentConfig = {
  id: string;
  name: string;
  model: string;              // e.g. "claude-sonnet-4-6"
  systemPrompt?: string;      // custom identity/instructions
  tools?: string[];            // allowlisted tool names
  maxSubagentDepth?: number;   // default 1
};

type AgentsConfig = {
  defaultAgentId: string;
  agents: AgentConfig[];
};
```

#### Step 1.3: Config loader (`src/config/loader.ts`)
- Read `agents.json` from project root
- Validate shape, resolve default agent
- Function: `loadConfig(): AgentsConfig`

#### Step 1.4: Session store (`src/sessions/session-store.ts`)
File-based conversation persistence (mirrors OpenClaw's SessionManager):
```typescript
type Message = {
  role: "user" | "assistant" | "tool_result";
  content: string | ToolResultContent[];
  toolCalls?: ToolCall[];
  timestamp: number;
};

type Session = {
  sessionId: string;
  agentId: string;
  messages: Message[];
  metadata: { createdAt: number; spawnDepth?: number; spawnedBy?: string };
};

// Functions:
// loadSession(sessionId): Session | null
// saveSession(session): void
// appendMessage(sessionId, message): void
// listSessions(): SessionSummary[]
```
Store as `sessions/<sessionId>.json`.

#### Step 1.5: Session key generation (`src/sessions/session-key.ts`)
```typescript
// Format: agent:<agentId>:<scope>
// Examples:
//   agent:main:cli           — interactive CLI session
//   agent:main:subagent:abc  — spawned subagent
function createSessionKey(agentId: string, scope: string): string
function parseSessionKey(key: string): { agentId: string; scope: string }
```

#### Step 1.6: System prompt builder (`src/agents/system-prompt.ts`)
Two modes, matching OpenClaw:
```typescript
function buildSystemPrompt(agent: AgentConfig, opts: {
  mode: "full" | "minimal";      // full for main, minimal for subagents
  availableTools: ToolDef[];
  depth?: number;
  maxDepth?: number;
  task?: string;                  // subagent task description
}): string
```

- **Full mode:** Complete identity, all tool docs, general instructions
- **Minimal mode:** "You are a subagent spawned for a specific task. Stay focused."

#### Step 1.7: Anthropic streaming wrapper (`src/runner/stream.ts`)
Direct replacement for OpenClaw's `streamSimple`:
```typescript
async function* streamAgentResponse(params: {
  apiKey: string;
  model: string;
  system: string;
  messages: AnthropicMessage[];
  tools?: AnthropicTool[];
  maxTokens?: number;
}): AsyncGenerator<StreamEvent>

type StreamEvent =
  | { type: "text"; text: string }
  | { type: "tool_use"; id: string; name: string; input: Record<string, unknown> }
  | { type: "done"; stopReason: string; usage: { input: number; output: number } }
```

#### Step 1.8: Tool definitions (`src/agents/tools.ts`)
Start with 2-3 simple built-in tools:
```typescript
type ToolDef = {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  execute: (input: Record<string, unknown>, ctx: ToolContext) => Promise<string>;
};

// Built-in tools:
// - "read_file": Read a file from workspace
// - "write_file": Write a file to workspace
// - "web_search": Simple web search (mock or real)
```

#### Step 1.9: Tool executor (`src/runner/tool-executor.ts`)
```typescript
async function executeToolCall(
  toolCall: { id: string; name: string; input: Record<string, unknown> },
  tools: ToolDef[],
  ctx: ToolContext,
): Promise<{ toolUseId: string; content: string; isError: boolean }>
```

#### Step 1.10: Agent runner — the core loop (`src/runner/runner.ts`)
This is the heart of the system. Mirrors OpenClaw's `runEmbeddedAttempt`:
```typescript
async function runAgent(params: {
  agentId: string;
  sessionId: string;
  prompt: string;
  config: AgentsConfig;
  apiKey: string;
}): Promise<AgentResult>

// The agentic loop:
// 1. Load session from disk
// 2. Build system prompt
// 3. Append user message to session
// 4. Call Anthropic API (streaming)
// 5. If response contains tool_use → execute tools → append results → goto 4
// 6. If response is end_turn → save session → return result
```

#### Step 1.11: CLI entry point (`src/index.ts`)
```bash
npx mini-agents chat --agent main "What files are in this directory?"
npx mini-agents agents list
npx mini-agents sessions list
```

#### Step 1.12: Tests
- `src/sessions/session-store.test.ts` — CRUD operations
- `src/runner/runner.test.ts` — Mock Anthropic client, verify tool loop
- `src/config/loader.test.ts` — Config validation

---

### Phase 2: The Lane System — Concurrent Agent Execution

**Goal:** Multiple agents can run concurrently without blocking each other.

#### Step 2.1: Command queue with lanes (`src/lanes/command-queue.ts`)
Port OpenClaw's lane system (simplified):
```typescript
type LaneState = {
  queue: QueueEntry[];
  running: boolean;
  maxConcurrent: number;   // 1 per lane by default
};

const lanes = new Map<string, LaneState>();

// Enqueue a task on a named lane
function enqueueOnLane<T>(lane: string, task: () => Promise<T>): Promise<T>

// Internal: drain the lane (run next task when current finishes)
function drainLane(lane: string): void
```

Key behavior:
- Each session gets its own lane: `session:<sessionId>`
- Within a lane: serial execution (one turn at a time)
- Across lanes: concurrent (async interleaving on event loop)

#### Step 2.2: Integrate lanes into runner
Wrap `runAgent()` calls in `enqueueOnLane()`:
```typescript
function runAgentOnLane(params: RunAgentParams): Promise<AgentResult> {
  const lane = `session:${params.sessionId}`;
  return enqueueOnLane(lane, () => runAgent(params));
}
```

#### Step 2.3: Tests
- Verify two agents on different lanes run concurrently
- Verify two messages to the same agent run serially
- Verify lane cleanup after task completion

---

### Phase 3: Subagents — The Core Coordination Pattern

**Goal:** Agents can spawn child agents for subtasks, with depth limits and result delivery.

#### Step 3.1: Depth tracking (`src/subagents/depth.ts`)
```typescript
const MAX_SPAWN_DEPTH = 2;  // configurable per agent

function getSpawnDepth(sessionId: string): number
// Reads from session metadata (spawnDepth field)

function canSpawnSubagent(sessionId: string): boolean
// Returns spawnDepth < maxSpawnDepth
```

#### Step 3.2: Subagent registry (`src/subagents/registry.ts`)
In-memory + disk-persisted tracking (mirrors OpenClaw's subagent-registry.ts):
```typescript
type SubagentRun = {
  runId: string;
  childSessionId: string;
  parentSessionId: string;
  task: string;
  mode: "run" | "session";   // ephemeral vs persistent
  status: "running" | "completed" | "error" | "timeout";
  startedAt: number;
  endedAt?: number;
  result?: string;
  error?: string;
};

const runs = new Map<string, SubagentRun>();

function registerRun(run: SubagentRun): void
function completeRun(runId: string, result: string): void
function failRun(runId: string, error: string): void
function listRunsForParent(parentSessionId: string): SubagentRun[]
function getActiveRunCount(parentSessionId: string): number
```

#### Step 3.3: Spawn function (`src/subagents/spawn.ts`)
The core spawn logic (mirrors OpenClaw's `spawnSubagentDirect`):
```typescript
async function spawnSubagent(params: {
  parentSessionId: string;
  task: string;
  agentId?: string;          // which agent config to use (default: same as parent)
  model?: string;            // override model for subagent
  mode?: "run" | "session";  // ephemeral or persistent
  timeoutMs?: number;        // max execution time
  config: AgentsConfig;
  apiKey: string;
}): Promise<SpawnResult>

// Flow:
// 1. Check depth limit (canSpawnSubagent)
// 2. Create child session with metadata { spawnDepth: parentDepth + 1, spawnedBy: parentId }
// 3. Build minimal system prompt
// 4. Enqueue child execution on its own lane
// 5. Register in subagent registry
// 6. Return { runId, childSessionId, status: "accepted" }
```

#### Step 3.4: Result announcement (`src/subagents/announce.ts`)
Deliver subagent results back to parent (mirrors OpenClaw's `runSubagentAnnounceFlow`):
```typescript
async function announceSubagentResult(params: {
  runId: string;
  parentSessionId: string;
  childSessionId: string;
  result: string;
  task: string;
}): Promise<void>

// Flow:
// 1. Load child session to get final output
// 2. Format completion message:
//    "[Subagent completed] Task: <task>\nResult: <result>"
// 3. Append as user message to parent's session
// 4. If parent agent is idle, the next prompt will see it
// 5. If parent is waiting (via tool call), resolve the tool result
```

#### Step 3.5: "spawn_subagent" tool
Add to the agent's tool set so the LLM can trigger spawning:
```typescript
{
  name: "spawn_subagent",
  description: "Spawn a subagent to handle a subtask independently. Returns immediately. Result will be delivered as a follow-up message.",
  inputSchema: {
    type: "object",
    properties: {
      task: { type: "string", description: "What the subagent should do" },
      model: { type: "string", description: "Optional model override" },
    },
    required: ["task"],
  },
  execute: async (input, ctx) => {
    const result = await spawnSubagent({
      parentSessionId: ctx.sessionId,
      task: input.task as string,
      model: input.model as string | undefined,
      config: ctx.config,
      apiKey: ctx.apiKey,
    });
    return `Subagent spawned. Run ID: ${result.runId}. Result will be delivered when complete.`;
  },
}
```

#### Step 3.6: "list_subagents" and "kill_subagent" tools
Management tools matching OpenClaw's `subagents-tool.ts`:
```typescript
// list_subagents: Show running/completed subagent runs for this session
// kill_subagent: Terminate a running subagent by runId
```

#### Step 3.7: Subagent wait mechanism
Two patterns, matching OpenClaw:

**Fire-and-forget (async):** Subagent runs on its own lane. When done, result is announced to parent. Parent sees it on next turn.

**Blocking wait (sync tool call):** Parent's tool call blocks until subagent completes:
```typescript
{
  name: "spawn_subagent_and_wait",
  description: "Spawn a subagent and wait for its result before continuing.",
  execute: async (input, ctx) => {
    const result = await spawnSubagent({ ...params, mode: "run" });
    // Wait for completion (with timeout)
    const outcome = await waitForSubagentCompletion(result.runId, timeoutMs);
    return outcome.result ?? `Subagent failed: ${outcome.error}`;
  },
}
```

#### Step 3.8: Tests
- Spawn a subagent, verify it runs on a separate lane
- Verify depth limit enforcement (depth 2 agent cannot spawn)
- Verify result announcement back to parent
- Verify timeout behavior
- Verify kill terminates the child

---

### Phase 4: HTTP Gateway (Optional Extension)

**Goal:** Expose the agent system over HTTP, so agents communicate via RPC (like OpenClaw's gateway).

#### Step 4.1: Express gateway server (`src/gateway/server.ts`)
```typescript
// POST /agent       — Run an agent turn
// POST /agent/wait  — Wait for a running agent to finish
// GET  /sessions    — List sessions
// GET  /agents      — List configured agents
// POST /subagent/kill — Terminate a subagent
```

#### Step 4.2: Refactor subagent spawn to use gateway RPC
Instead of directly calling `runAgent()`, subagents call the gateway:
```typescript
// Before (Phase 3): direct function call
await runAgent({ ... });

// After (Phase 4): HTTP RPC (matches OpenClaw's callGateway pattern)
await fetch("http://localhost:3000/agent", {
  method: "POST",
  body: JSON.stringify({ agentId, sessionId, prompt }),
});
```

This decouples spawning from execution, exactly like OpenClaw does.

---

## Implementation Order & Time Estimates

| Step | What | Depends on |
|------|------|-----------|
| 1.1 | Project scaffold | — |
| 1.2-1.3 | Config types + loader | 1.1 |
| 1.4-1.5 | Session store + keys | 1.1 |
| 1.6 | System prompt builder | 1.2 |
| 1.7 | Anthropic streaming | 1.1 |
| 1.8-1.9 | Tools + executor | 1.1 |
| 1.10 | Agent runner (core loop) | 1.4, 1.6, 1.7, 1.8 |
| 1.11 | CLI | 1.10 |
| 1.12 | Phase 1 tests | 1.10 |
| 2.1-2.3 | Lane system | 1.10 |
| 3.1 | Depth tracking | 1.4 |
| 3.2 | Subagent registry | 2.1 |
| 3.3 | Spawn function | 3.1, 3.2, 1.10 |
| 3.4 | Result announcement | 3.2, 3.3 |
| 3.5-3.6 | Subagent tools | 3.3, 3.4 |
| 3.7 | Wait mechanism | 3.3 |
| 3.8 | Phase 3 tests | 3.5 |
| 4.1-4.2 | HTTP gateway | 3.5 |

## How to Run

```bash
# Set your API key
export ANTHROPIC_API_KEY=sk-ant-...

# Install
npm install

# Build
npm run build

# Interactive chat with default agent
npx mini-agents chat "Hello, what can you do?"

# Chat with specific agent
npx mini-agents chat --agent researcher "Find info about TypeScript 5.9"

# List agents
npx mini-agents agents list

# List sessions
npx mini-agents sessions list

# Run tests
npm test
```

## Example `agents.json`

```json
{
  "defaultAgentId": "main",
  "agents": [
    {
      "id": "main",
      "name": "Assistant",
      "model": "claude-sonnet-4-6",
      "systemPrompt": "You are a helpful assistant. You can delegate research tasks to subagents.",
      "tools": ["read_file", "write_file", "spawn_subagent", "spawn_subagent_and_wait", "list_subagents"],
      "maxSubagentDepth": 2
    },
    {
      "id": "researcher",
      "name": "Research Assistant",
      "model": "claude-haiku-4-5-20251001",
      "systemPrompt": "You are a focused research assistant. Complete the assigned task efficiently.",
      "tools": ["read_file", "web_search"],
      "maxSubagentDepth": 0
    }
  ]
}
```

## Key Concepts Mapped: OpenClaw → mini-agents

| OpenClaw concept | mini-agents equivalent | Simplification |
|-----------------|----------------------|----------------|
| `@mariozechner/pi-ai` streamSimple | `@anthropic-ai/sdk` streaming | Direct SDK instead of wrapper library |
| `SessionManager` (JSONL) | `session-store.ts` (JSON files) | Single JSON file per session instead of JSONL |
| `pi-embedded-runner/run.ts` (1400 LOC) | `runner.ts` (~200 LOC) | No retry/fallback/auth-profile logic |
| `command-queue.ts` lanes | `command-queue.ts` lanes | Same pattern, fewer features |
| `subagent-spawn.ts` + `callGateway` | `spawn.ts` + direct function call | Skip HTTP RPC in Phase 3, add in Phase 4 |
| `subagent-registry.ts` (1168 LOC) | `registry.ts` (~100 LOC) | No announce retry, orphan reconciliation |
| `subagent-announce.ts` (1025 LOC) | `announce.ts` (~50 LOC) | Simple message append, no retry logic |
| `system-prompt.ts` (705 LOC) | `system-prompt.ts` (~80 LOC) | Two modes, no skills/memory/sandbox sections |
| `openclaw-tools.ts` (50+ tools) | `tools.ts` (5 tools) | Just file I/O, search, and subagent management |
| Channel bindings + routing | CLI only | No multi-channel complexity |
| Plugin system + extensions | None | Out of scope |
| Auth profiles + model fallback | Single API key | No fallback chain |
