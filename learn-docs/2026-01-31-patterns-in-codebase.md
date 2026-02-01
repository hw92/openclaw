# Patterns in the OpenClaw Codebase

Mapping architecture patterns to actual source code implementations with file paths and line numbers.

> **Note**: The project was renamed from "clawdbot" to "moltbot" to "openclaw".

---

## 1. Session Lanes (Promise Mutex)

**File**: `src/process/command-queue.ts`

```typescript
// Lines 9-24: Queue entry and lane state
type QueueEntry = {
  task: () => Promise<unknown>;
  resolve: (value: unknown) => void;
  reject: (reason?: unknown) => void;
  enqueuedAt: number;
  warnAfterMs: number;
  onWait?: (waitMs: number, queuedAhead: number) => void;
};

type LaneState = {
  lane: string;
  queue: QueueEntry[];
  active: number;
  maxConcurrent: number;
  draining: boolean;
};

// Lines 44-90: The pump mechanism - pure Promise-based serialization
function drainLane(lane: string) {
  const state = getLaneState(lane);
  state.draining = true;

  const pump = () => {
    while (state.active < state.maxConcurrent && state.queue.length > 0) {
      const entry = state.queue.shift() as QueueEntry;
      state.active += 1;
      void (async () => {
        try {
          const result = await entry.task();
          state.active -= 1;
          pump();  // <-- Recursive drain
          entry.resolve(result);
        } catch (err) {
          state.active -= 1;
          pump();
          entry.reject(err);
        }
      })();
    }
    state.draining = false;
  };
  pump();
}
```

**Key insight**: No Redis, no external deps — just a `Map<string, LaneState>` with Promise chaining.

---

## 2. Tool Policy (8-Layer Filtering)

**File**: `src/agents/tool-policy.ts`

```typescript
// Lines 1-6: Profile types
export type ToolProfileId = "minimal" | "coding" | "messaging" | "full";

// Lines 13-57: Tool groups (expandable macros)
export const TOOL_GROUPS: Record<string, string[]> = {
  "group:memory": ["memory_search", "memory_get"],
  "group:web": ["web_search", "web_fetch"],
  "group:fs": ["read", "write", "edit", "apply_patch"],
  "group:runtime": ["exec", "process"],
  "group:sessions": ["sessions_list", "sessions_history", "sessions_send", ...],
  "group:ui": ["browser", "canvas"],
  "group:automation": ["cron", "gateway"],
  "group:messaging": ["message"],
  "group:nodes": ["nodes"],
  "group:openclaw": [/* all native tools */],
};

// Lines 59-76: Profiles define allow/deny per use case
const TOOL_PROFILES: Record<ToolProfileId, ToolProfilePolicy> = {
  minimal: { allow: ["session_status"] },
  coding:  { allow: ["group:fs", "group:runtime", "group:sessions", "group:memory", "image"] },
  messaging: { allow: ["group:messaging", "sessions_list", "sessions_history", ...] },
  full: {},  // No restrictions
};
```

**Key insight**: `group:*` macros expand to tool lists. Profiles compose these groups.

---

## 3. Node Capability Advertisement

**File**: `src/gateway/node-registry.ts`

```typescript
// Lines 5-22: NodeSession with capabilities
export type NodeSession = {
  nodeId: string;
  connId: string;
  client: GatewayWsClient;
  displayName?: string;
  platform?: string;
  caps: string[];      // <-- CAPABILITIES (e.g. ["camera.snap", "location.get"])
  commands: string[];  // <-- COMMANDS the node can execute
  permissions?: Record<string, boolean>;
  connectedAtMs: number;
};

// Lines 44-79: Registration extracts caps from connect payload
register(client: GatewayWsClient, opts: { remoteIp?: string }) {
  const connect = client.connect;
  const caps = Array.isArray(connect.caps) ? connect.caps : [];
  const commands = Array.isArray(connect.commands) ? connect.commands : [];

  const session: NodeSession = {
    nodeId,
    caps,      // iPhone advertises: ["camera.snap", "location.get"]
    commands,  // Available commands
    ...
  };
  this.nodesById.set(nodeId, session);
  return session;
}
```

**Key insight**: Gateway maintains `nodesById` map. When agent calls `camera.snap`, Gateway looks up which node has that capability.

---

## 4. Gateway Hub (WebSocket Server)

**File**: `src/gateway/server/ws-connection.ts`

```typescript
// Lines 21-43: Central attachment point for all WS connections
export function attachGatewayWsConnectionHandler(params: {
  wss: WebSocketServer;
  clients: Set<GatewayWsClient>;  // <-- All connected clients
  broadcast: (event: string, payload: unknown, opts?) => void;  // <-- Hub broadcasts
  ...
}) {
  wss.on("connection", (socket, upgradeReq) => {
    const connId = randomUUID();
    // ... handshake, auth validation
    // Client added to clients Set
    // Messages routed through message-handler.ts
  });
}
```

**Key insight**: Single `WebSocketServer` instance. All clients (CLI, iOS, Android, Web) connect here. `broadcast()` pushes events to all subscribers.

---

## 5. TypeBox Schema (Avoids anyOf)

**File**: `src/agents/schema/typebox.ts`

```typescript
// Lines 13-24: Custom string enum that avoids anyOf/oneOf
// NOTE: Avoid Type.Union([Type.Literal(...)]) which compiles to anyOf.
// Some providers reject anyOf in tool schemas; a flat string enum is safer.
export function stringEnum<T extends readonly string[]>(
  values: T,
  options: StringEnumOptions<T> = {},
) {
  return Type.Unsafe<T[number]>({
    type: "string",
    enum: [...values],  // <-- Flat enum, not Union
    ...options,
  });
}

export function optionalStringEnum<T extends readonly string[]>(values: T, options = {}) {
  return Type.Optional(stringEnum(values, options));
}
```

**Key insight**: Some LLM providers (Google) reject `anyOf` in tool schemas. This helper generates flat `enum` instead.

---

## 6. Bootstrap Files (SOUL.md, AGENTS.md, etc.)

**File**: `src/agents/workspace.ts`

```typescript
// Lines 20-29: Bootstrap file constants
export const DEFAULT_AGENT_WORKSPACE_DIR = resolveDefaultAgentWorkspaceDir();
export const DEFAULT_AGENTS_FILENAME = "AGENTS.md";
export const DEFAULT_SOUL_FILENAME = "SOUL.md";
export const DEFAULT_TOOLS_FILENAME = "TOOLS.md";
export const DEFAULT_IDENTITY_FILENAME = "IDENTITY.md";
export const DEFAULT_USER_FILENAME = "USER.md";
export const DEFAULT_HEARTBEAT_FILENAME = "HEARTBEAT.md";
export const DEFAULT_BOOTSTRAP_FILENAME = "BOOTSTRAP.md";
export const DEFAULT_MEMORY_FILENAME = "MEMORY.md";

// Lines 9-18: Workspace resolution (supports profiles)
export function resolveDefaultAgentWorkspaceDir(env = process.env, homedir = os.homedir) {
  const profile = env.OPENCLAW_PROFILE?.trim();
  if (profile && profile.toLowerCase() !== "default") {
    return path.join(homedir(), ".openclaw", `workspace-${profile}`);
  }
  return path.join(homedir(), ".openclaw", "workspace");
}
```

**Key insight**: Files loaded from `~/.openclaw/workspace/`. Multiple profiles supported via `OPENCLAW_PROFILE` env var.

---

## 7. A2A Communication (sessions_send)

**File**: `src/agents/tools/sessions-send-tool.ts`

```typescript
// Lines 31-37: TypeBox schema for the tool
const SessionsSendToolSchema = Type.Object({
  sessionKey: Type.Optional(Type.String()),
  label: Type.Optional(Type.String({ minLength: 1, maxLength: SESSION_LABEL_MAX_LENGTH })),
  agentId: Type.Optional(Type.String({ minLength: 1, maxLength: 64 })),
  message: Type.String(),
  timeoutSeconds: Type.Optional(Type.Number({ minimum: 0 })),
});

// Lines 39-50: Tool definition
export function createSessionsSendTool(opts?) {
  return {
    label: "Session Send",
    name: "sessions_send",
    description: "Send a message into another session...",
    parameters: SessionsSendToolSchema,
    execute: async (_toolCallId, args) => {
      // ... policy checks, then:
      return runSessionsSendA2AFlow({
        targetSessionKey: params.sessionKey,
        message: params.message,
        // Supports ping-pong turns for multi-agent conversation
      });
    }
  };
}
```

**Key insight**: Agent A can send message to Agent B's session. Supports `maxPingPongTurns` for back-and-forth A2A dialogue.

---

## 8. Adaptive Compaction

**File**: `src/agents/compaction.ts`

```typescript
// Lines 7-9: Compaction constants
export const BASE_CHUNK_RATIO = 0.4;
export const MIN_CHUNK_RATIO = 0.15;
export const SAFETY_MARGIN = 1.2;  // 20% buffer for token estimation

// Lines 16-18: Token estimation
export function estimateMessagesTokens(messages: AgentMessage[]): number {
  return messages.reduce((sum, msg) => sum + estimateTokens(msg), 0);
}

// Lines 27-66: Split by token share (adaptive chunking)
export function splitMessagesByTokenShare(messages: AgentMessage[], parts = 2): AgentMessage[][] {
  const totalTokens = estimateMessagesTokens(messages);
  const targetTokens = totalTokens / normalizedParts;
  const chunks: AgentMessage[][] = [];
  let current: AgentMessage[] = [];
  let currentTokens = 0;

  for (const message of messages) {
    const messageTokens = estimateTokens(message);
    // Start new chunk when exceeding target
    if (chunks.length < normalizedParts - 1 &&
        current.length > 0 &&
        currentTokens + messageTokens > targetTokens) {
      chunks.push(current);
      current = [];
      currentTokens = 0;
    }
    current.push(message);
    currentTokens += messageTokens;
  }
  return chunks;
}
```

**Key insight**: History split into chunks by token count, then each chunk summarized. Keeps recent messages intact.

---

## 9. Block Streaming

**File**: `src/auto-reply/reply/block-streaming.ts`

```typescript
// Lines 12-14: Default streaming thresholds
const DEFAULT_BLOCK_STREAM_MIN = 800;
const DEFAULT_BLOCK_STREAM_MAX = 1200;
const DEFAULT_BLOCK_STREAM_COALESCE_IDLE_MS = 1000;

// Lines 52-57: Coalescing config type
export type BlockStreamingCoalescing = {
  minChars: number;   // Don't send until this many chars
  maxChars: number;   // Force send at this threshold
  idleMs: number;     // Send after idle period
  joiner: string;     // How to join coalesced blocks
};

// Lines 59-67: Resolution with channel-specific limits
export function resolveBlockStreamingChunking(cfg, provider?, accountId?) {
  return {
    minChars: ...,
    maxChars: ...,
    breakPreference: "paragraph" | "newline" | "sentence",
  };
}
```

**Key insight**: Streaming coalesces small chunks before sending. Different channels have different limits (Telegram: 4000, Discord: 2000).

---

## 10. Command Lane Types

**File**: `src/process/lanes.ts`

```typescript
export const enum CommandLane {
  Main = "main",
  Cron = "cron",
  Subagent = "subagent",
  Nested = "nested",
}
```

**File**: `src/gateway/server-lanes.ts`

```typescript
export function applyGatewayLaneConcurrency(cfg: ReturnType<typeof loadConfig>) {
  setCommandLaneConcurrency(CommandLane.Cron, cfg.cron?.maxConcurrentRuns ?? 1);
  setCommandLaneConcurrency(CommandLane.Main, resolveAgentMaxConcurrent(cfg));
  setCommandLaneConcurrency(CommandLane.Subagent, resolveSubagentMaxConcurrent(cfg));
}
```

**Key insight**: Different lanes have different concurrency limits. Main lane serializes user requests; cron/subagent lanes can run in parallel.

---

## 11. Skills Loading

**File**: `src/agents/skills/workspace.ts`

```typescript
import {
  formatSkillsForPrompt,
  loadSkillsFromDir,
  type Skill,
} from "@mariozechner/pi-coding-agent";

import { parseFrontmatter, resolveOpenClawMetadata } from "./frontmatter.js";

function filterSkillEntries(entries, config?, skillFilter?, eligibility?) {
  return entries.filter((entry) => shouldIncludeSkill({ entry, config, eligibility }));
}

function sanitizeSkillCommandName(raw: string): string {
  // Discord command validation (≤100 char descriptions)
  return raw.toLowerCase()
    .replace(/[^a-z0-9_]+/g, "_")
    .slice(0, SKILL_COMMAND_MAX_LENGTH);
}
```

**File**: `src/agents/skills/frontmatter.ts`

```typescript
import JSON5 from "json5";

export function parseFrontmatter(content: string): ParsedSkillFrontmatter {
  return parseFrontmatterBlock(content);
}

export type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  id?: string;
  bins?: string[];
  os?: string[];
  // ...
};
```

**Key insight**: Skills are markdown files with YAML frontmatter. Frontmatter defines dependencies and permissions; body teaches the agent how to use the skill.

---

## 12. Bootstrap File Loading

**File**: `src/agents/bootstrap-files.ts`

```typescript
export async function resolveBootstrapFilesForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  agentId?: string;
}): Promise<WorkspaceBootstrapFile[]> {
  const bootstrapFiles = filterBootstrapFilesForSession(
    await loadWorkspaceBootstrapFiles(params.workspaceDir),
    sessionKey,
  );
  return applyBootstrapHookOverrides({
    files: bootstrapFiles,
    workspaceDir: params.workspaceDir,
    config: params.config,
    sessionKey: params.sessionKey,
    agentId: params.agentId,
  });
}
```

**File**: `src/agents/bootstrap-hooks.ts`

```typescript
export async function applyBootstrapHookOverrides(params) {
  const context: AgentBootstrapHookContext = {
    workspaceDir: params.workspaceDir,
    bootstrapFiles: params.files,
    cfg: params.config,
    sessionKey: params.sessionKey,
    agentId,
  };

  const event = createInternalHookEvent("agent", "bootstrap", sessionKey, context);
  await triggerInternalHook(event);  // <-- Plugins can modify bootstrap files
  return event.context.bootstrapFiles;
}
```

**Key insight**: Bootstrap files can be modified by hooks before agent run. Plugins can inject context dynamically.

---

## Summary: Pattern → File Mapping

| Pattern | File | Key Lines |
|---------|------|-----------|
| **Session Lanes** | `src/process/command-queue.ts` | 9-90 |
| **Lane Types** | `src/process/lanes.ts` | 1-6 |
| **Lane Concurrency** | `src/gateway/server-lanes.ts` | 1-10 |
| **Tool Policy** | `src/agents/tool-policy.ts` | 1-120 |
| **Node Registry** | `src/gateway/node-registry.ts` | 5-100 |
| **Gateway Hub** | `src/gateway/server/ws-connection.ts` | 21-80 |
| **TypeBox Helpers** | `src/agents/schema/typebox.ts` | 1-44 |
| **Bootstrap Files** | `src/agents/workspace.ts` | 9-29 |
| **Bootstrap Loading** | `src/agents/bootstrap-files.ts` | 1-60 |
| **Bootstrap Hooks** | `src/agents/bootstrap-hooks.ts` | 1-32 |
| **A2A Send Tool** | `src/agents/tools/sessions-send-tool.ts` | 31-80 |
| **A2A Flow** | `src/agents/tools/sessions-send-tool.a2a.ts` | 19-80 |
| **A2A List Tool** | `src/agents/tools/sessions-list-tool.ts` | 21-60 |
| **Compaction** | `src/agents/compaction.ts` | 7-66 |
| **Block Streaming** | `src/auto-reply/reply/block-streaming.ts` | 12-80 |
| **Skills Workspace** | `src/agents/skills/workspace.ts` | 1-100 |
| **Skills Frontmatter** | `src/agents/skills/frontmatter.ts` | 1-80 |

---

## Directory Structure Reference

```
src/
├── agents/
│   ├── workspace.ts          # Bootstrap file constants
│   ├── bootstrap-files.ts    # Bootstrap loading
│   ├── bootstrap-hooks.ts    # Hook overrides
│   ├── compaction.ts         # Context compression
│   ├── tool-policy.ts        # 8-layer filtering
│   ├── schema/
│   │   └── typebox.ts        # Schema helpers
│   ├── skills/
│   │   ├── workspace.ts      # Skill loading
│   │   └── frontmatter.ts    # YAML parsing
│   └── tools/
│       ├── sessions-send-tool.ts      # A2A send
│       ├── sessions-send-tool.a2a.ts  # A2A flow
│       └── sessions-list-tool.ts      # A2A discovery
├── gateway/
│   ├── node-registry.ts      # Node capabilities
│   ├── server-lanes.ts       # Lane concurrency
│   └── server/
│       └── ws-connection.ts  # WebSocket hub
├── process/
│   ├── command-queue.ts      # Promise mutex
│   └── lanes.ts              # Lane enum
└── auto-reply/
    └── reply/
        └── block-streaming.ts  # Streaming config
```

---

*Written by Claude (Opus 4.5) | 2026-01-31 PST*
