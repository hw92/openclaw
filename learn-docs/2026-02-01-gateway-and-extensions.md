# OpenClaw Gateway and Extensions Guide

A beginner-friendly guide to understanding the Gateway architecture and the different types of extensions in OpenClaw.

**Last updated:** 2026-02-01

---

## Table of Contents

1. [What is the Gateway?](#what-is-the-gateway)
2. [What are Extensions?](#what-are-extensions)
3. [Extension Categories](#extension-categories)
4. [Channel Extensions](#channel-extensions)
5. [Auth Provider Extensions](#auth-provider-extensions)
6. [Tool Extensions](#tool-extensions)
7. [Memory Extensions](#memory-extensions)
8. [Utility Extensions](#utility-extensions)
9. [How Extensions Work Together](#how-extensions-work-together)
10. [Real-World Scenarios](#real-world-scenarios)
11. [Extension Technical Details](#extension-technical-details)
12. [Summary](#summary)

---

## What is the Gateway?

The **Gateway** is the heart of OpenClaw - it's the central server that coordinates everything.

```
┌─────────────────────────────────────────────────────────────────┐
│                         GATEWAY                                  │
│                    (src/gateway/)                                │
│                                                                  │
│   The "brain" that:                                             │
│   • Receives messages from all channels                         │
│   • Routes messages to AI providers                             │
│   • Manages sessions and context                                │
│   • Executes AI tool calls                                      │
│   • Sends responses back                                        │
│   • Schedules tasks (cron)                                      │
│   • Loads and manages extensions                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Think of the Gateway as an **air traffic controller** - it doesn't fly the planes (AI providers) or run the airports (messaging platforms), but it coordinates all the traffic between them.

### Gateway Core Functions

| Function | Description |
|----------|-------------|
| **Message Routing** | Receives messages, routes to AI, sends responses |
| **Session Management** | Tracks conversation history and context |
| **Extension Loading** | Loads and manages all plugins |
| **Tool Execution** | Runs tools when AI requests them |
| **Scheduling** | Handles cron jobs and timed tasks |
| **WebSocket Server** | Provides real-time communication |

---

## What are Extensions?

**Extensions** are plugins that add functionality to the Gateway. They use a **generic plugin system** but serve **different purposes**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTENSION = PLUGIN                            │
│                                                                  │
│   A package that:                                                │
│   • Lives in extensions/ folder                                  │
│   • Has its own package.json                                     │
│   • Implements OpenClawPluginAPI                                 │
│   • Gets loaded by the Gateway                                   │
│   • Can add channels, tools, auth, memory, commands, etc.       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Insight: Same System, Different Purposes

All extensions use the **same plugin mechanism**, but they serve **completely different roles**:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   google-antigravity-auth  ──→  Gets OAuth tokens for AI       │
│                                                                  │
│   imessage                 ──→  Sends/receives iMessages        │
│                                                                  │
│   voice-call               ──→  Makes phone calls               │
│                                                                  │
│   memory-lancedb           ──→  Stores AI memories              │
│                                                                  │
│   All are "extensions" but do VERY different things!            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Extension Categories

Extensions fall into **five main categories**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTENSION CATEGORIES                          │
└─────────────────────────────────────────────────────────────────┘

  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  CHANNELS   │  │    AUTH     │  │   TOOLS     │
  │             │  │  PROVIDERS  │  │             │
  │ How users   │  │ How to get  │  │ What AI     │
  │ reach AI    │  │ AI access   │  │ can do      │
  └─────────────┘  └─────────────┘  └─────────────┘

  ┌─────────────┐  ┌─────────────┐
  │   MEMORY    │  │  UTILITIES  │
  │             │  │             │
  │ How AI      │  │ Other       │
  │ remembers   │  │ enhancements│
  └─────────────┘  └─────────────┘
```

### Quick Reference Table

| Category | Purpose | Examples | When Active |
|----------|---------|----------|-------------|
| **Channels** | Message I/O with platforms | telegram, discord, imessage | Always listening |
| **Auth Providers** | Get credentials for AI services | google-antigravity-auth | On login/token refresh |
| **Tools** | Give AI new abilities | voice-call, llm-task | When AI decides to use |
| **Memory** | Store/retrieve memories | memory-lancedb | During conversations |
| **Utilities** | Various enhancements | diagnostics-otel, lobster | Various |

---

## Channel Extensions

**Purpose:** Connect the Gateway to messaging platforms so users can chat with the AI.

### How Channels Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHANNEL EXTENSION FLOW                        │
└─────────────────────────────────────────────────────────────────┘

    External Service                    Gateway
    ════════════════                    ═══════

    ┌──────────────┐                ┌──────────────┐
    │   Telegram   │                │              │
    │   Servers    │ ──webhook──→   │   Telegram   │
    │              │                │  Extension   │
    │              │ ←──API call──  │              │
    └──────────────┘                └──────┬───────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │   Gateway    │
                                    │    Core      │
                                    └──────┬───────┘
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │     AI       │
                                    │   Provider   │
                                    └──────────────┘
```

### Available Channel Extensions

| Extension | Platform | Notes |
|-----------|----------|-------|
| `telegram` | Telegram | Bot API |
| `discord` | Discord | Bot integration |
| `slack` | Slack | Workspace app |
| `imessage` | Apple iMessage | macOS only |
| `matrix` | Matrix protocol | Decentralized |
| `msteams` | Microsoft Teams | Enterprise |
| `googlechat` | Google Chat | Workspace |
| `whatsapp` | WhatsApp | Via Baileys |
| `line` | LINE | Popular in Asia |
| `bluebubbles` | iMessage (alt) | Via BlueBubbles server |
| `twitch` | Twitch chat | Streaming |
| `tlon` | Tlon/Urbit | Decentralized |
| `mattermost` | Mattermost | Self-hosted |
| `nextcloud-talk` | Nextcloud Talk | Self-hosted |
| `nostr` | Nostr protocol | Decentralized |
| `zalo` | Zalo | Vietnam |
| `zalouser` | Zalo (user) | Personal account |

### Channel Extension Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHAT A CHANNEL EXTENSION DOES                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CONNECT                                                      │
│     • Authenticate with the platform                             │
│     • Establish connection (WebSocket, polling, webhook)         │
│     • Handle reconnection on failures                            │
│                                                                  │
│  2. RECEIVE                                                      │
│     • Listen for incoming messages                               │
│     • Parse platform-specific format                             │
│     • Convert to Gateway's internal format                       │
│     • Handle media (images, voice, files)                        │
│                                                                  │
│  3. SEND                                                         │
│     • Receive response from Gateway                              │
│     • Convert to platform-specific format                        │
│     • Handle markdown/formatting differences                     │
│     • Send via platform API                                      │
│                                                                  │
│  4. FEATURES                                                     │
│     • Group chat support                                         │
│     • Typing indicators                                          │
│     • Read receipts                                              │
│     • Reactions                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Auth Provider Extensions

**Purpose:** Get OAuth tokens/credentials so OpenClaw can use external AI services.

### How Auth Providers Work

```
┌─────────────────────────────────────────────────────────────────┐
│                  AUTH PROVIDER EXTENSION FLOW                    │
└─────────────────────────────────────────────────────────────────┘

    You run: openclaw models auth login --provider google-antigravity
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Auth Extension (google-antigravity-auth)│
    │                                          │
    │   1. Generate OAuth URL                  │
    │   2. Open browser                        │
    │   3. You log in with Google              │
    │   4. Receive callback with code          │
    │   5. Exchange code for tokens            │
    │   6. Store tokens securely               │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Tokens stored in:                      │
    │   ~/.openclaw/credentials/               │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Now Gateway can use Google's AI APIs   │
    │   with your credentials                  │
    └──────────────────────────────────────────┘
```

### Available Auth Provider Extensions

| Extension | Service | What it enables |
|-----------|---------|-----------------|
| `google-antigravity-auth` | Google Cloud Code Assist | Use Claude via Google |
| `google-gemini-cli-auth` | Google Gemini | Use Gemini models |
| `minimax-portal-auth` | MiniMax AI | Use MiniMax models |
| `qwen-portal-auth` | Qwen/Alibaba | Use Qwen models |
| `copilot-proxy` | GitHub Copilot | Use Copilot's API |

### Why Auth Extensions Exist

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Problem: Some AI services require OAuth, not just API keys    │
│                                                                  │
│   ┌─────────────────┐    vs    ┌─────────────────┐              │
│   │ Simple API Key  │          │ OAuth Required  │              │
│   │                 │          │                 │              │
│   │ Anthropic       │          │ Google Cloud    │              │
│   │ OpenAI          │          │ GitHub Copilot  │              │
│   │ OpenRouter      │          │ Some enterprise │              │
│   │                 │          │ services        │              │
│   │ Just paste key  │          │ Need OAuth flow │              │
│   │ in config       │          │ (browser login) │              │
│   └─────────────────┘          └─────────────────┘              │
│                                                                  │
│   Auth extensions handle the complex OAuth dance for you!       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tool Extensions

**Purpose:** Give the AI new abilities to interact with the real world.

### How Tool Extensions Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    TOOL EXTENSION FLOW                           │
└─────────────────────────────────────────────────────────────────┘

    User: "Call Jenny and wish her happy birthday"
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   AI decides to use voice_call tool      │
    │                                          │
    │   Tool call: {                           │
    │     "tool": "voice_call",                │
    │     "action": "initiate_call",           │
    │     "to": "+1555123456",                 │
    │     "message": "Happy birthday Jenny!"   │
    │   }                                      │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Gateway routes to voice-call extension │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   voice-call extension:                  │
    │   1. Calls Twilio/Telnyx API             │
    │   2. Initiates real phone call           │
    │   3. Streams TTS audio                   │
    │   4. Returns result to Gateway           │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   AI receives result, responds to user:  │
    │   "I've called Jenny and wished her      │
    │    happy birthday!"                      │
    └──────────────────────────────────────────┘
```

### Available Tool Extensions

| Extension | Tool Name | What AI Can Do |
|-----------|-----------|----------------|
| `voice-call` | `voice_call` | Make/receive phone calls |
| `llm-task` | `llm_task` | Delegate tasks to other LLMs |

### Built-in Tools (Not Extensions)

The Gateway also has built-in tools that don't require extensions:

| Tool | Capability |
|------|------------|
| `browser` | Control web browser |
| `canvas` | Visual canvas for diagrams |
| `cron` | Schedule future tasks |
| `message` | Send messages to channels |
| `tts` | Text-to-speech |
| `web_search` | Search the web |
| `web_fetch` | Fetch web pages |
| `image` | Generate/process images |
| `sessions_*` | Manage chat sessions |

### Tool Extension vs Built-in Tool

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Built-in Tools              Tool Extensions                   │
│   ══════════════              ═══════════════                   │
│                                                                  │
│   • Part of core Gateway      • Separate packages               │
│   • Always available          • Must be enabled                 │
│   • In src/agents/tools/      • In extensions/                  │
│   • Can't be removed          • Can be added/removed            │
│                                                                  │
│   Examples:                   Examples:                         │
│   - web_search                - voice_call                      │
│   - cron                      - llm_task                        │
│   - browser                   - (future: email, calendar...)    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Memory Extensions

**Purpose:** Provide different backends for AI memory storage.

### How Memory Extensions Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY EXTENSION FLOW                         │
└─────────────────────────────────────────────────────────────────┘

    During conversation:

    AI: "I'll remember that your favorite color is blue"
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Gateway calls memory extension         │
    │   memory.store({                         │
    │     key: "user_preference",              │
    │     value: "favorite color is blue"      │
    │   })                                     │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Memory extension stores data:          │
    │                                          │
    │   memory-core:                           │
    │     → Simple JSON file                   │
    │                                          │
    │   memory-lancedb:                        │
    │     → Vector database (semantic search)  │
    └──────────────────────────────────────────┘

    Later, in another conversation:

    User: "What's my favorite color?"
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Gateway queries memory extension       │
    │   memory.search("favorite color")        │
    └──────────────────────────────────────────┘
                              │
                              ▼
    ┌──────────────────────────────────────────┐
    │   Memory extension returns:              │
    │   "favorite color is blue"               │
    └──────────────────────────────────────────┘
                              │
                              ▼
    AI: "Your favorite color is blue!"
```

### Available Memory Extensions

| Extension | Storage Type | Best For |
|-----------|--------------|----------|
| `memory-core` | Simple key-value | Basic needs |
| `memory-lancedb` | Vector database | Semantic search, large memory |

### Why Different Memory Backends?

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   memory-core (simple)           memory-lancedb (advanced)      │
│   ════════════════════           ═════════════════════════      │
│                                                                  │
│   • JSON file storage            • Vector embeddings            │
│   • Exact key lookup             • Semantic similarity search   │
│   • Fast, lightweight            • "Find similar memories"      │
│   • Good for small data          • Good for large knowledge     │
│                                                                  │
│   Query: "blue"                  Query: "what color do I like"  │
│   Finds: key="blue" only         Finds: "favorite color is blue"│
│                                  (understands meaning!)         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Utility Extensions

**Purpose:** Various enhancements that don't fit other categories.

### Available Utility Extensions

| Extension | Purpose |
|-----------|---------|
| `lobster` | UI/UX enhancements, terminal styling |
| `open-prose` | Writing/prose skills for AI |
| `diagnostics-otel` | OpenTelemetry monitoring/tracing |

---

## How Extensions Work Together

Here's a complete scenario showing multiple extension types working together:

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLETE SCENARIO: Schedule Birthday Call           │
└─────────────────────────────────────────────────────────────────┘

    You (via Telegram): "Remember Jenny's birthday is Feb 14.
                         Call her at 9am and wish her happy birthday."

    ════════════════════════════════════════════════════════════════

    STEP 1: Message received via CHANNEL extension
    ───────────────────────────────────────────────

    ┌────────────┐         ┌────────────┐
    │  Telegram  │ ──────→ │  telegram  │ ──────→ Gateway
    │  Servers   │         │ extension  │
    └────────────┘         └────────────┘

    ════════════════════════════════════════════════════════════════

    STEP 2: AI processes with credentials from AUTH extension
    ──────────────────────────────────────────────────────────

    Gateway ──────→ AI Provider (Claude)
                    (using tokens from google-antigravity-auth
                     or direct Anthropic API key)

    ════════════════════════════════════════════════════════════════

    STEP 3: AI stores memory via MEMORY extension
    ──────────────────────────────────────────────

    AI: "I'll remember Jenny's birthday"

    Gateway ──────→ memory-lancedb extension
                    stores: "Jenny birthday Feb 14"

    ════════════════════════════════════════════════════════════════

    STEP 4: AI schedules task via built-in cron tool
    ─────────────────────────────────────────────────

    AI uses cron tool: schedule call for Feb 14, 9am

    ════════════════════════════════════════════════════════════════

    STEP 5: On Feb 14 at 9am, cron triggers
    ───────────────────────────────────────

    Cron ──────→ Wakes AI agent

    ════════════════════════════════════════════════════════════════

    STEP 6: AI uses TOOL extension to make call
    ────────────────────────────────────────────

    AI ──────→ voice-call extension ──────→ Twilio ──────→ Jenny's phone

    Jenny answers, hears: "Happy birthday from [You]!"

    ════════════════════════════════════════════════════════════════

    STEP 7: AI reports back via CHANNEL extension
    ──────────────────────────────────────────────

    Gateway ──────→ telegram extension ──────→ Telegram ──────→ You

    "I called Jenny and wished her happy birthday! She said thank you."
```

### Extension Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         GATEWAY                                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Extension Manager                     │   │
│   │                                                          │   │
│   │    Loads and coordinates all extension types             │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│          │              │              │              │          │
│          ▼              ▼              ▼              ▼          │
│   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐   │
│   │ CHANNELS  │  │   AUTH    │  │  TOOLS    │  │  MEMORY   │   │
│   │           │  │           │  │           │  │           │   │
│   │ telegram  │  │ google-   │  │ voice-    │  │ memory-   │   │
│   │ discord   │  │ antigrav  │  │ call      │  │ lancedb   │   │
│   │ imessage  │  │ gemini    │  │ llm-task  │  │           │   │
│   │ slack     │  │ copilot   │  │           │  │           │   │
│   │ ...       │  │ ...       │  │ ...       │  │           │   │
│   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘   │
│         │              │              │              │          │
└─────────┼──────────────┼──────────────┼──────────────┼──────────┘
          │              │              │              │
          ▼              ▼              ▼              ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
    │ Messaging │  │    AI     │  │   Real    │  │ Database  │
    │ Platforms │  │ Providers │  │   World   │  │  Storage  │
    └───────────┘  └───────────┘  └───────────┘  └───────────┘
```

---

## Real-World Scenarios

### Scenario 1: Receive Telegram, Respond via AI

```
Extensions used:
├── telegram (CHANNEL) - receive/send messages
└── (none for auth if using Anthropic API key directly)

Flow:
Telegram → telegram extension → Gateway → Claude → Gateway → telegram extension → Telegram
```

### Scenario 2: Use Google's Claude via OAuth

```
Extensions used:
├── telegram (CHANNEL) - receive/send messages
└── google-antigravity-auth (AUTH) - OAuth for Google Cloud

Flow:
1. First: openclaw models auth login --provider google-antigravity
2. Then: Telegram → Gateway → Claude (via Google) → Gateway → Telegram
```

### Scenario 3: AI Makes Phone Call

```
Extensions used:
├── discord (CHANNEL) - receive command
├── voice-call (TOOL) - make phone call
└── memory-lancedb (MEMORY) - remember contact info

Flow:
Discord message → Gateway → AI decides to call → voice-call extension → Twilio → Phone
```

### Scenario 4: Multi-Channel Response

```
Extensions used:
├── telegram (CHANNEL) - receive original message
├── slack (CHANNEL) - send copy to work channel
└── imessage (CHANNEL) - send copy to personal

Flow:
Telegram → Gateway → AI → responds to Telegram
                       → also posts to Slack
                       → also sends to iMessage
```

---

## Extension Technical Details

### Extension File Structure

```
extensions/
└── my-extension/
    ├── package.json           # Package manifest
    ├── openclaw.plugin.json   # Plugin metadata
    ├── index.ts               # Main entry point
    ├── README.md              # Documentation
    └── src/                   # Additional source files
        ├── handlers.ts
        └── utils.ts
```

### Plugin API Interface

All extensions can implement parts of this interface:

```typescript
export interface OpenClawPluginAPI {
  // ═══════════════════════════════════════════════════════════
  // FOR CHANNEL EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  gateway?: {
    // RPC methods (called via WebSocket)
    rpc?: Record<string, Function>;

    // HTTP endpoints (webhooks, callbacks)
    http?: Record<string, Function>;

    // Lifecycle hooks
    onStart?: () => Promise<void>;
    onStop?: () => Promise<void>;
  };

  // ═══════════════════════════════════════════════════════════
  // FOR TOOL EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  tools?: Tool[];  // Tools available to AI

  // ═══════════════════════════════════════════════════════════
  // FOR AUTH EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  auth?: AuthProvider[];  // OAuth providers

  // ═══════════════════════════════════════════════════════════
  // FOR MEMORY EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  memory?: MemoryProvider;  // Storage backend

  // ═══════════════════════════════════════════════════════════
  // FOR CLI EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  commands?: Command[];  // CLI commands

  // ═══════════════════════════════════════════════════════════
  // FOR SKILL EXTENSIONS
  // ═══════════════════════════════════════════════════════════
  skills?: string[];  // AI skill definitions
}
```

### Extension Loading Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTENSION LOADING                             │
└─────────────────────────────────────────────────────────────────┘

    Gateway starts
         │
         ▼
    ┌─────────────────────────────────────┐
    │ 1. Scan extension directories:      │
    │    • extensions/ (bundled)          │
    │    • ~/.openclaw/extensions/ (user) │
    └─────────────────────────────────────┘
         │
         ▼
    ┌─────────────────────────────────────┐
    │ 2. Read openclaw.plugin.json        │
    │    or package.json openclaw field   │
    └─────────────────────────────────────┘
         │
         ▼
    ┌─────────────────────────────────────┐
    │ 3. Check if enabled in config       │
    │    plugins.entries.<name>.enabled   │
    └─────────────────────────────────────┘
         │
         ▼
    ┌─────────────────────────────────────┐
    │ 4. Load via jiti (TypeScript)       │
    │    No build step needed!            │
    └─────────────────────────────────────┘
         │
         ▼
    ┌─────────────────────────────────────┐
    │ 5. Register with Gateway:           │
    │    • Add tools to AI                │
    │    • Add RPC handlers               │
    │    • Add HTTP routes                │
    │    • Initialize channel connections │
    └─────────────────────────────────────┘
```

### Enabling/Disabling Extensions

```bash
# Enable an extension
openclaw plugins enable voice-call

# Disable an extension
openclaw plugins disable voice-call

# List all extensions
openclaw plugins list

# Install from npm
openclaw plugins install @openclaw/voice-call
```

Or via config (`~/.openclaw/openclaw.json`):

```json5
{
  plugins: {
    entries: {
      "voice-call": { enabled: true },
      "telegram": { enabled: true },
      "discord": { enabled: false },  // disabled
    }
  }
}
```

---

## Summary

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   EXTENSION = Generic Plugin Mechanism                          │
│                                                                  │
│   Same package format, but DIFFERENT purposes:                  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                          │   │
│   │   CHANNELS     →  How users reach the AI                │   │
│   │                   (Telegram, Discord, iMessage...)       │   │
│   │                                                          │   │
│   │   AUTH         →  How to access AI providers             │   │
│   │                   (Google OAuth, Copilot tokens...)      │   │
│   │                                                          │   │
│   │   TOOLS        →  What AI can do in the world           │   │
│   │                   (Phone calls, email, calendar...)      │   │
│   │                                                          │   │
│   │   MEMORY       →  How AI remembers things               │   │
│   │                   (Simple storage, vector DB...)         │   │
│   │                                                          │   │
│   │   UTILITIES    →  Other enhancements                    │   │
│   │                   (Monitoring, UI improvements...)       │   │
│   │                                                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   All loaded by the GATEWAY and work together!                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **Gateway** is the brain - coordinates everything
2. **Extensions** are plugins - add capabilities
3. **Same system, different purposes** - channels, auth, tools, memory, utilities
4. **Flexible** - easy to add new extensions for new services
5. **Work together** - a single request might use multiple extension types

---

## Learning Resources

Related learn-docs:
- `2026-02-01-project-structure-overview.md` - Overall project structure
- `2026-02-01-installation-and-configuration.md` - Building and configuring

Official docs:
- Plugin development: https://docs.openclaw.ai/plugin
- Channel setup: https://docs.openclaw.ai/channels

---

*This guide was created to help developers understand the Gateway architecture and the different types of extensions in OpenClaw.*
