# Moltbot Architecture Deep Dive

A comprehensive guide to understanding the moltbot codebase.

> **Note**: The project was renamed from "clawdbot" to "moltbot" in January 2026.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Agent System](#2-agent-system)
3. [Messaging Channels](#3-messaging-channels)
4. [Gateway](#4-gateway)
5. [Plugin System](#5-plugin-system)
6. [Providers](#6-providers)
7. [Companion Apps](#7-companion-apps)
8. [Key Files Reference](#8-key-files-reference)

---

## 1. Overview

### What is Moltbot?

Moltbot is a **personal AI assistant platform** that runs locally on your devices and bridges an AI agent with all major messaging platforms.

**Problem it solves:**
- Unified AI assistant across WhatsApp, Telegram, Discord, Slack, Signal, iMessage, and more
- Local-first, under user control
- Always-on voice capabilities and live Canvas interfaces
- Multi-device interaction (macOS, iOS, Android)

### Architecture (Hub-and-Spoke Model)

```
Messaging Channels ────┐
(WhatsApp/Telegram/    │
 Discord/Slack/etc.)   ▼
                  ┌─────────────────┐
                  │    GATEWAY      │ (ws://127.0.0.1:18789)
                  │   (WS server)   │
                  └────────┬────────┘
                           │
              ┌────────────┼────────────┬────────────┐
              ▼            ▼            ▼            ▼
          Pi Agent      CLI         WebChat     macOS/iOS/Android
```

### Directory Structure

| Path | Purpose |
|------|---------|
| `src/gateway/` | Central WebSocket server, message routing |
| `src/channels/` | Channel plugin infrastructure |
| `src/agents/` | Pi agent runtime, tools, auth |
| `src/commands/` | CLI command implementations |
| `src/providers/` | Model providers (Claude, OpenAI, etc.) |
| `src/plugins/` | Plugin loader and registry |
| `extensions/` | Optional channel/feature plugins |
| `apps/` | macOS, iOS, Android companion apps |
| `docs/` | Mintlify documentation |

### Key Entry Points

- `moltbot gateway` — Start the daemon
- `moltbot agent --message "..."` — Invoke agent directly
- `moltbot send` — Send messages
- `moltbot onboard` — Interactive setup wizard
- `moltbot doctor` — Diagnostics

### Tech Stack

- **Runtime:** Node 22+, TypeScript (ESM), Bun supported
- **Agent:** Pi agent packages (`@mariozechner/pi-*`)
- **Web:** Express, Hono, WebSocket
- **Build:** pnpm, Vitest, oxlint

---

## 2. Agent System

The agent system is built on **Pi** — an embedded AI agent framework from `@mariozechner/pi-*` packages that runs directly in the gateway process.

### Core Flow

```
Client Request → Gateway → Pi Agent Runtime → Streaming Response → Channel Delivery
```

### Agent Invocation

**Entry points:**
- CLI: `moltbot agent --message "..."`
- Gateway WebSocket: Messages from channels route to `server-methods/agent.ts`

**Execution function** (`src/agents/pi-embedded-runner/run.ts`):
```typescript
runEmbeddedPiAgent(params) → EmbeddedPiRunResult
```

### Bootstrap Files (Agent Identity)

Agents load markdown files from a workspace directory (default: `~/.moltbot/agent/`):

| File | Purpose |
|------|---------|
| `SOUL.md` | Agent identity/personality |
| `AGENTS.md` | Workspace rules and conventions |
| `TOOLS.md` | Local tool notes, device preferences |
| `USER.md` | Info about the user being served |
| `MEMORY.md` | Long-term curated memories (main session only) |
| `BOOTSTRAP.md` | First-run setup (deleted after use) |

### Tools Available

| Category | Tools | Location |
|----------|-------|----------|
| **File/Code** | `read`, `write`, `edit`, `apply_patch` | pi-coding-agent |
| **Exec** | `exec` (shell commands) | `src/agents/bash-tools.ts` |
| **Moltbot** | `session_status`, `memory_*`, `cron`, `gateway` | `src/agents/moltbot-tools.ts` |
| **Channels** | `telegram`, `discord`, `slack`, `whatsapp`, `signal` | `src/agents/channel-tools.ts` |
| **Web** | `web_search`, `web_fetch`, `browser_tool` | `src/agents/tools/` |
| **Advanced** | `canvas_host`, `image_tool`, `voice`, `nodes_tool` | various |

**Tool Policy** (`src/agents/pi-tools.policy.ts`):
- Allow/deny lists per session, group, or subagent
- Subagents inherit parent policy with restrictions

### Session & Transcript Management

**Storage format:** JSONL (one JSON object per line)
- Location: `~/.moltbot/sessions/<session-key>.jsonl`
- Managed by Pi's `SessionManager`

**Features:**
- Write locks prevent concurrent corruption
- Transcript repair for malformed entries
- Compaction summarizes old messages to save tokens

### Response Streaming

**Event handlers** (`src/agents/pi-embedded-subscribe.ts`):
```
message_start → message_update (chunks) → message_end
tool_execution_start → tool_execution_update → tool_execution_end
```

**Callbacks:**
- `onPartialReply` — streaming text chunks
- `onBlockReply` — formatted message blocks
- `onToolResult` — tool call outcomes

### Agent Execution Lifecycle

```
1. Gateway validates request, resolves session file
2. Acquire session write lock
3. Load bootstrap files (SOUL.md, AGENTS.md, etc.)
4. Resolve model + auth (with failover)
5. Initialize tools (filtered by policy)
6. Build system prompt
7. Run Pi agent loop:
   ├─ Send to LLM
   ├─ Parse tool calls
   ├─ Execute tools
   ├─ Add results to conversation
   └─ Repeat until stop_reason = "end_turn"
8. Stream responses to channels
9. Persist transcript to JSONL
10. Release lock, return result
```

---

## 3. Messaging Channels

Moltbot uses a **plugin-based adapter architecture** where each messaging platform implements a standard `ChannelPlugin` interface.

### Channel Plugin Structure

```typescript
type ChannelPlugin<ResolvedAccount> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;

  // Adapters (each handles one aspect)
  config: ChannelConfigAdapter;      // Read/write config
  outbound: ChannelOutboundAdapter;  // Send messages
  gateway: ChannelGatewayAdapter;    // Monitor lifecycle
  security: ChannelSecurityAdapter;  // DM allowlist policy
  groups: ChannelGroupAdapter;       // Group-specific behavior
  pairing: ChannelPairingAdapter;    // User approval flow
  threading: ChannelThreadingAdapter; // Reply/thread handling
  // ... 16+ adapter types total
};
```

### Built-in Channels

| Channel | Location | Protocol |
|---------|----------|----------|
| Telegram | `src/telegram/` | grammY bot API |
| WhatsApp | `src/web/` | Baileys (Web automation) |
| Discord | `src/discord/` | discord.js |
| Slack | `src/slack/` | Bolt (Socket Mode) |
| Signal | `src/signal/` | signal-cli REST |
| iMessage | `src/imessage/` | osascript/imsg CLI |
| Google Chat | `src/channels/plugins/googlechat.ts` | HTTP webhooks |

### Extension Channels (in `extensions/`)

Matrix, Microsoft Teams, BlueBubbles, Zalo, Mattermost

### Message Flow

```
INBOUND:
Platform message → Monitor (polls/subscribes) → Normalize to MessageContext
    → resolveAgentRoute() → Check policies → Dispatch to agent

OUTBOUND:
Agent reply → deliverOutboundPayloads() → Load outbound adapter
    → Chunk text (if needed) → sendText/sendMedia → Platform API
```

### Routing (`src/routing/resolve-route.ts`)

Messages routed to agents based on **binding priority**:

1. **Peer binding** — specific DM/group participant
2. **Guild/team binding** — Discord server or Slack workspace
3. **Account binding** — specific channel account
4. **Wildcard** — `accountId: "*"`
5. **Default agent** — fallback

```yaml
# Example config
session:
  bindings:
    - match:
        channel: telegram
        peer: { kind: dm, id: "123456789" }
      agentId: alice
    - match:
        channel: discord
        guildId: "server-id"
      agentId: bob
```

### Key Adapters

**ChannelOutboundAdapter** — Sends messages:
```typescript
{
  deliveryMode: "direct" | "gateway" | "hybrid";
  textChunkLimit?: number;
  sendText: (ctx) => Promise<Result>;
  sendMedia: (ctx) => Promise<Result>;
}
```

**ChannelSecurityAdapter** — DM policies:
- `"open"` — accept DMs from anyone
- `"pairing"` — users must approve first (default)
- `"allowlist"` — only configured users

### Chunking & Streaming

**Text chunking** (`src/auto-reply/chunk.ts`):
- `chunkText()` — simple length-based split
- `chunkByNewline()` — break on line boundaries
- `chunkByParagraph()` — break on blank lines

**Per-channel limits:**

| Channel | Default Limit |
|---------|---------------|
| Telegram | 4000 chars |
| Discord | 2000 chars |
| WhatsApp | 65536 chars |

### Group Chat Handling

**Mention requirements:**
```yaml
channels:
  telegram:
    groups:
      "-1001234567890":
        requireMention: true
      "*":
        requireMention: false
```

**Per-sender tool policies:**
```yaml
channels:
  discord:
    guilds:
      "server-id":
        channels:
          "#general":
            toolPolicy:
              default: [chat]
              senders:
                "admin-id": [chat, exec]
```

---

## 4. Gateway

The gateway is the central WebSocket server that coordinates all communication.

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                      GATEWAY SERVER                         │
│                  (ws://127.0.0.1:18789)                     │
├─────────────────────────────────────────────────────────────┤
│  HTTP Server(s)  │  WebSocket Server  │  Event Broadcaster  │
├──────────────────┼────────────────────┼─────────────────────┤
│  RPC Handlers    │  Channel Manager   │  Lane Queue         │
├──────────────────┼────────────────────┼─────────────────────┤
│  Node Registry   │  Health State      │  Dedupe Cache       │
└─────────────────────────────────────────────────────────────┘
```

### Startup Flow (`src/gateway/server.impl.ts`)

1. Load and validate configuration
2. Initialize agent/subagent registries
3. Load gateway plugins
4. Create HTTP/WebSocket servers
5. Initialize mDNS discovery
6. Start maintenance timers (health, keepalive, cleanup)
7. Start channel monitors
8. Start sidecars (canvas host, browser control)

### WebSocket Protocol

**Frame Types:**
```typescript
// Request
{ type: "req", id: string, method: string, params?: object }

// Response
{ type: "res", id: string, ok: boolean, result?: unknown, error?: ErrorShape }

// Event
{ type: "event", event: string, payload: unknown, seq: number }
```

**Connection Handshake:**
```
Client connects → Server sends challenge (nonce)
    → Client sends connect request (auth + identity)
    → Server validates → Connect OK/FAIL
    → Bidirectional RPC + events
```

### RPC Methods (~85 core methods)

| Category | Methods |
|----------|---------|
| **Health/Status** | `health`, `status`, `channels.status`, `usage.*` |
| **Agent** | `agent`, `agent.wait`, `agent.identity.get`, `agents.list` |
| **Chat** | `chat.send`, `chat.abort`, `chat.history` |
| **Config** | `config.get`, `config.set`, `config.patch`, `config.schema` |
| **Sessions** | `sessions.list`, `sessions.preview`, `sessions.reset` |
| **Devices** | `device.pair.*`, `device.token.*` |
| **Nodes** | `node.pair.*`, `node.invoke`, `node.rename` |
| **Cron** | `cron.list`, `cron.add`, `cron.run`, `cron.remove` |
| **Channels** | `channels.logout`, `send` |

**Authorization by scope:**
- `operator.read` — health, logs, status
- `operator.write` — send, agent, chat
- `operator.admin` — config, wizard, updates
- `operator.pairing` — device/node pairing
- `operator.approvals` — exec approvals

### Lane-Based Concurrency

Tasks queued into **lanes** with configurable concurrency:

| Lane | Purpose | Default |
|------|---------|---------|
| `main` | Agent replies | `config.agent.maxConcurrent` |
| `cron` | Scheduled jobs | 1 |
| `subagent` | Nested agent calls | configurable |

### Event Broadcasting (`src/gateway/server-broadcast.ts`)

**Core events:**
- `agent` — agent started/thinking/done/error
- `chat` — delta/final/error
- `presence` — client connected/disconnected
- `health` — system health snapshot
- `tick` — keepalive (~30s)

### Authentication

**Three auth methods:**
1. **Token auth** — shared secret (`gateway.auth.token`)
2. **Password auth** — shared secret (`gateway.auth.password`)
3. **Device signature** — EdDSA signed payload with nonce

### Health & Maintenance

**Health snapshot** (refreshed every 5 minutes):
- Agent status & recent runs
- Channel status per account
- Skill registrations
- Config issues

**Maintenance timers:**

| Timer | Interval | Purpose |
|-------|----------|---------|
| Tick | ~30s | Keepalive broadcast |
| Health | ~5min | Refresh health snapshot |
| Dedupe cleanup | 1min | Prune expired entries |

### Node Coordination

**Node invocation (async RPC):**
```typescript
await nodeRegistry.invoke({
  nodeId: "ios-node",
  command: "camera.capture",
  params: { ... },
  timeoutMs: 30000
})
```

**Subscription-based event routing:**
```typescript
subscribe(nodeId, sessionKey)
sendToSession(sessionKey, event, payload)
```

---

## 5. Plugin System

Moltbot uses a **modular, registration-based plugin architecture**.

### Plugin Flow

```
Discovery → Manifest Loading → Module Loading → Registration → Lifecycle
```

### Plugin Manifest (`moltbot.plugin.json`)

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "What it does",
  "version": "1.0.0",
  "kind": "memory",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string" }
    }
  },
  "uiHints": {
    "apiKey": { "label": "API Key", "sensitive": true }
  }
}
```

### Plugin Definition

```typescript
import type { MoltbotPluginApi } from "moltbot/plugin-sdk";

const plugin = {
  id: "my-plugin",
  name: "My Plugin",

  register(api: MoltbotPluginApi) {
    api.registerTool(myTool);
    api.registerChannel({ plugin: myChannelPlugin });
    api.on("message_received", handleMessage);
  }
};

export default plugin;
```

### Plugin SDK (`MoltbotPluginApi`)

```typescript
type MoltbotPluginApi = {
  // Identification
  id: string;
  name: string;
  source: string;

  // Configuration
  config: MoltbotConfig;
  pluginConfig?: object;

  // Runtime & logging
  runtime: PluginRuntime;
  logger: PluginLogger;

  // Registration methods
  registerTool(tool, opts?): void;
  registerHook(events, handler, opts?): void;
  registerHttpHandler(handler): void;
  registerHttpRoute(params): void;
  registerChannel(registration): void;
  registerGatewayMethod(method, handler): void;
  registerCli(registrar, opts?): void;
  registerService(service): void;
  registerProvider(provider): void;
  registerCommand(command): void;

  // Modern hook API
  on<K>(hookName, handler, opts?): void;
};
```

### Plugin Capabilities

| Capability | Method | Purpose |
|------------|--------|---------|
| **Tools** | `registerTool()` | Agent tools |
| **Hooks** | `on()` | Lifecycle events |
| **Channels** | `registerChannel()` | Messaging platforms |
| **Gateway Methods** | `registerGatewayMethod()` | RPC endpoints |
| **HTTP Routes** | `registerHttpRoute()` | Webhooks |
| **CLI Commands** | `registerCli()` | CLI commands |
| **Services** | `registerService()` | Background services |
| **Providers** | `registerProvider()` | LLM providers |

### Lifecycle Hooks

```typescript
// Agent lifecycle
api.on("before_agent_start", async (event, ctx) => { });
api.on("agent_end", async (event, ctx) => { });

// Message lifecycle
api.on("message_received", async (event, ctx) => { });
api.on("message_sending", async (event, ctx) => { });
api.on("message_sent", async (event, ctx) => { });

// Tool execution
api.on("before_tool_call", async (event, ctx) => { });
api.on("after_tool_call", async (event, ctx) => { });

// Session & memory
api.on("session_start", async (event, ctx) => { });
api.on("before_compaction", async (event, ctx) => { });

// Gateway
api.on("gateway_start", async (event, ctx) => { });
api.on("gateway_stop", async (event, ctx) => { });
```

### Plugin Discovery

**Search paths (in order):**
1. **Bundled**: `src/plugins/` + `extensions/`
2. **Global**: `~/.moltbot/plugins/`
3. **Workspace**: `${workspace}/.moltbot/plugins/`
4. **Config**: paths in `plugins.load.paths`

### Plugin Configuration

```yaml
plugins:
  enabled: true
  allow: ["whitelist-ids"]
  deny: ["blacklist-ids"]
  slots:
    memory: "memory-lancedb"
  load:
    paths: ["/custom/path"]
  entries:
    my-plugin:
      enabled: true
      config:
        apiKey: "..."
```

### Key Patterns

1. **Isolation** — Each plugin gets its own scope
2. **Lazy loading** — Jiti loads modules on demand (supports TS)
3. **Memory slot** — Only ONE memory plugin active at a time
4. **Hook priority** — Hooks support ordering
5. **Config validation** — JSON Schema validates before loading

---

## 6. Providers

The providers system manages LLM model endpoints with multi-provider support, failover, and credential management.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Model Catalog Layer (Discovery)                 │
│  - Model metadata (id, name, API type, context window)      │
│  - Vision capability detection                              │
│  - Reasoning/thinking support flags                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              Provider Configuration Layer                    │
│  - BaseUrl, API version (anthropic-messages, openai-*, etc)│
│  - Custom model definitions (models.json)                   │
│  - Auth mode (api-key, oauth, token, aws-sdk)              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│            Auth Profile & Credential Layer                   │
│  - Auth Profile Store (auth-profiles.json)                  │
│  - Multiple credentials per provider (round-robin)          │
│  - Usage stats, cooldown tracking, failure reasons          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│           Model Selection & Fallback Layer                   │
│  - Primary/fallback model chains                            │
│  - Alias resolution                                         │
└─────────────────────────────────────────────────────────────┘
```

### Built-in Providers

| Provider | API Type | Auth |
|----------|----------|------|
| Anthropic (Claude) | `anthropic-messages` | API key |
| OpenAI (GPT) | `openai-completions` | API key |
| Google (Gemini) | `google-generative-ai` | API key |
| Groq | `openai-completions` | API key |
| Mistral | `openai-completions` | API key |
| AWS Bedrock | `bedrock-converse-stream` | AWS SDK |
| GitHub Copilot | `openai-responses` | OAuth token exchange |

### Implicit Providers (Auto-discovered)

- **MiniMax**: if `MINIMAX_API_KEY` exists
- **Moonshot/Kimi**: if `MOONSHOT_API_KEY` or `KIMICODE_API_KEY` exists
- **Ollama**: if local `http://127.0.0.1:11434` accessible
- **Venice**: if `VENICE_API_KEY` exists

### Provider Configuration

```yaml
models:
  mode: "merge"  # merge with implicit providers
  providers:
    anthropic: {}  # Uses pi-ai defaults
    openai:
      baseUrl: "https://api.openai.com/v1"
      apiKey: "${OPENAI_API_KEY}"
      api: "openai-completions"
    my-custom-provider:
      baseUrl: "https://my-api.example.com/v1"
      apiKey: "MY_API_KEY"
      api: "openai-completions"
      models:
        - id: "my-model-v1"
          name: "My Model V1"
          contextWindow: 128000
          maxTokens: 8192
```

### Auth Profile Store

Supports **multiple credentials per provider** with round-robin rotation:

```typescript
type AuthProfileStore = {
  profiles: Record<string, AuthProfileCredential>;
  order?: Record<string, string[]>;        // Provider → profile order
  lastGood?: Record<string, string>;       // Provider → last working profile
  usageStats?: Record<string, ProfileUsageStats>;
};

type AuthProfileCredential =
  | { type: "api_key"; provider: string; key: string }
  | { type: "token"; provider: string; token: string; expires?: number }
  | { type: "oauth"; provider: string; access: string; refresh?: string };
```

**Profile ID format**: `{provider}:{identifier}` (e.g., `anthropic:default`, `openai:personal`)

### Auth Resolution Order

1. Explicit profile ID (if provided)
2. Custom provider config (`models.providers[].apiKey`)
3. Auth profile order override
4. Environment variable (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.)
5. Auth profile store (round-robin: oldest used first)
6. AWS SDK chain (for Bedrock)

### Model Selection & Fallback

**Model reference format**: `{provider}/{modelId}` (e.g., `anthropic/claude-opus-4-5`)

**Aliases** in config:
```yaml
agents:
  defaults:
    models:
      anthropic/claude-opus-4-5:
        alias: "op"
    model:
      primary: "op"
      fallbacks: ["openai/gpt-4o", "google/gemini-3-flash"]
```

**Fallback chain**: `Primary → Fallback 1 → Fallback 2 → Error`

### Failover & Error Handling

**Failover reasons:**
- `auth` — HTTP 401/403
- `billing` — HTTP 402
- `rate_limit` — HTTP 429
- `timeout` — HTTP 408, network errors
- `format` — Invalid response format
- `unknown` — Won't trigger fallback

**Cooldown tracking:**
- Failed auth: disabled until cooldown expires
- Rate limit: provider skipped during cooldown
- Automatic cooldown lift when expiry passes

---

## 7. Companion Apps

Moltbot uses a **distributed client-node-gateway architecture** with companion apps on macOS, iOS, and Android.

### Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   macOS     │     │    iOS      │     │   Android   │
│  Menu Bar   │     │    Node     │     │    Node     │
│  + Gateway  │     │   (Client)  │     │   (Client)  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │         WebSocket (Protocol v3)       │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────▼──────┐
                    │   GATEWAY   │
                    │  (Central)  │
                    └─────────────┘
```

### macOS App (Menu Bar + Gateway)

**Location**: `apps/macos/`

**Components:**
- Menu bar app (SwiftUI)
- `moltbot-mac` CLI binary
- Embedded gateway process management

**Features:**
- Starts/stops local gateway as subprocess
- Voice wake (Swabble speech recognition)
- Talk mode (ElevenLabs TTS)
- Canvas display in menu bar popup
- Auto-updates via Sparkle

**Key files:**
- `GatewayProcessManager.swift` — Process lifecycle
- `AppState.swift` — Settings persistence

### iOS App (Node Client)

**Location**: `apps/ios/`

**Architecture:**
- Companion node connecting to gateway via WebSocket
- Shared session key `main` across all clients
- SwiftUI with `@Observable` state management

**Capabilities:**

| Feature | Supported |
|---------|-----------|
| Chat | Main session |
| Canvas | WebView tab |
| Camera | Snap + Clip |
| Voice Wake | Built-in |
| Talk Mode | Voice I/O |
| Location | GPS |
| Screen | WebView capture |

**Node Commands:**
```
canvas.present / canvas.hide / canvas.navigate / canvas.eval
camera.list / camera.snap / camera.clip
location.get
system.notify / system.run
voice.wake
```

### Android App (Foreground Service Node)

**Location**: `apps/android/`

**Architecture:**
- Kotlin + Jetpack Compose
- Foreground service for persistent connection
- minSdk 31 (Android 12+)

**Key Components:**
- `NodeRuntime.kt` — Singleton managing all operations
- `NodeForegroundService.kt` — Always-on notification + connection

**Capabilities:**

| Feature | Supported |
|---------|-----------|
| Chat | Main session |
| Canvas | Compose UI |
| Camera | Snap + Clip |
| Voice Wake | Built-in |
| Talk Mode | Voice I/O |
| Location | GPS |
| Screen | MediaProjection |
| SMS | SmsManager |

**Permissions:**
```
INTERNET, CAMERA, RECORD_AUDIO, ACCESS_FINE_LOCATION,
POST_NOTIFICATIONS, SEND_SMS, NEARBY_WIFI_DEVICES
```

### Shared Code: MoltbotKit

**Location**: `apps/shared/MoltbotKit/`

**Swift Package modules:**
- **MoltbotProtocol** — Frame types, RPC params, events
- **MoltbotKit** — WebSocket connection, node commands

**Protocol Version 3 Frames:**
```typescript
// Request: client → gateway
{ type: "req", id: string, method: string, params?: object }

// Response: gateway → client
{ type: "res", id: string, ok: boolean, payload?: object, error?: object }

// Event: gateway → clients (broadcast)
{ type: "event", event: string, payload: object, seq: number }
```

**Connection Flow:**
```
1. Client connects WebSocket
2. Gateway sends challenge (nonce)
3. Client sends hello with auth + capabilities
4. Gateway responds hello-ok with features
5. Bidirectional RPC + event streaming
```

### Node Capability Comparison

| Feature | macOS | iOS | Android |
|---------|-------|-----|---------|
| **Gateway Host** | Yes | No | No |
| **Chat** | Via gateway | Main session | Main session |
| **Canvas** | Menu bar | Tab | Compose |
| **Camera** | No | Yes | Yes |
| **Voice Wake** | Swabble | Built-in | Built-in |
| **Talk Mode** | TTS | Voice I/O | Voice I/O |
| **Location** | No | GPS | GPS |
| **Screen Record** | No | WebView | MediaProjection |
| **SMS** | No | No | Yes |
| **Background** | N/A | Limited | Foreground service |

### A2UI System (Canvas-to-Agent)

When users interact with canvas UI elements:
1. JavaScript captures click/form submission
2. Sends tagged message: `action=`, `session=`, `component=`, `context=`
3. Gateway routes to agent
4. Agent processes and responds

---

## 8. Key Files Reference

### Agent System

| File | Purpose |
|------|---------|
| `src/agents/pi-embedded-runner/run.ts` | Main execution entry |
| `src/agents/bootstrap-files.ts` | Loads SOUL.md, AGENTS.md, etc. |
| `src/agents/pi-tools.ts` | Tool initialization |
| `src/agents/pi-embedded-subscribe.ts` | Response streaming |
| `src/gateway/server-methods/agent.ts` | Gateway handler |

### Channels

| File | Purpose |
|------|---------|
| `src/channels/plugins/types.plugin.ts` | ChannelPlugin interface |
| `src/channels/registry.ts` | Channel lookup & registration |
| `src/routing/resolve-route.ts` | Message → agent routing |
| `src/infra/outbound/deliver.ts` | Outbound delivery pipeline |
| `src/auto-reply/dispatch.ts` | Inbound message dispatch |
| `src/auto-reply/chunk.ts` | Text chunking algorithms |

### Gateway

| File | Purpose |
|------|---------|
| `src/gateway/server.impl.ts` | Main server implementation |
| `src/gateway/server-methods.ts` | RPC method dispatch |
| `src/gateway/server-methods/` | Individual method handlers |
| `src/gateway/server-channels.ts` | Channel lifecycle |
| `src/gateway/server-broadcast.ts` | Event broadcasting |
| `src/gateway/process/command-queue.ts` | Lane queue |
| `src/gateway/auth.ts` | Authentication logic |
| `src/gateway/node-registry.ts` | Node management |

### Plugin System

| File | Purpose |
|------|---------|
| `src/plugin-sdk/index.ts` | Plugin SDK exports |
| `src/plugins/loader.ts` | Module loading |
| `src/plugins/discovery.ts` | Plugin finding |
| `src/plugins/registry.ts` | Central registry |
| `src/plugins/hooks.ts` | Hook system |
| `extensions/*/` | Bundled extension plugins |

### Providers

| File | Purpose |
|------|---------|
| `src/agents/model-catalog.ts` | Model discovery and catalog |
| `src/agents/models-config.providers.ts` | Provider configuration |
| `src/agents/auth-profiles/` | Credential management |
| `src/agents/model-selection.ts` | Model resolution |
| `src/agents/model-fallback.ts` | Fallback chain handling |
| `src/agents/failover-error.ts` | Error classification |

### Companion Apps

| File | Purpose |
|------|---------|
| `apps/macos/Sources/` | macOS menu bar app |
| `apps/macos/Sources/Gateway/` | Gateway process management |
| `apps/ios/Sources/` | iOS node app |
| `apps/android/app/src/` | Android node app |
| `apps/shared/MoltbotKit/` | Shared Swift code |
| `apps/shared/MoltbotKit/Sources/MoltbotProtocol/` | Protocol types |

---

*Written by Claude (Opus 4.5) | 2026-01-26 14:15 PST*
*Updated 2026-01-27: Renamed from clawdbot to moltbot*
