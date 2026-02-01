# Crash Course: OpenClaw Architecture Patterns

A practical deep-dive into each key pattern with code examples and rationale.

> **Note**: The project was renamed from "clawdbot" to "moltbot" to "openclaw".

---

## 1. Design Philosophy

### Sovereign AI
**What**: User owns and controls all data physically. No cloud dependency for core data.

**How it works**:
```
~/.openclaw/
├── agent/
│   ├── SOUL.md      # Agent personality (you edit directly)
│   ├── USER.md      # Info about you
│   └── MEMORY.md    # Long-term memories
├── sessions/        # Conversation history (JSONL files)
└── config.yaml      # All settings
```

**Why it matters**: Delete a file = agent forgets. No vendor lock-in. Works offline. You can `grep` your agent's memories.

---

### OS-as-Surface
**What**: The operating system IS the AI interface. CLI tools > custom APIs.

**Traditional approach**:
```typescript
// Build custom audio processing library
import { AudioProcessor } from './lib/audio';
await audioProcessor.convert(file, 'mp3');
```

**OS-as-Surface approach**:
```typescript
// Just call ffmpeg
await exec('ffmpeg -i input.wav output.mp3');
```

**Why it matters**:
- LLMs already know shell commands from training data
- Zero coupling — swap any tool
- Agent can write NEW scripts to fill capability gaps
- Infinite extensibility without code changes

---

### Gateway-first
**What**: Single WebSocket server is the "brain stem" — all clients connect to it.

```
        WhatsApp ──┐
        Telegram ──┼──► Gateway (ws://127.0.0.1:18789) ◄──┬── CLI
        Discord ───┤         │                            ├── iOS app
        Slack ─────┘         │                            └── Web UI
                             ▼
                      Single Source of Truth
                    (sessions, state, routing)
```

**Why it matters**:
- Start conversation on phone, continue on desktop — same context
- One place to observe/debug all traffic
- Consistent auth, rate limiting, policies

---

### Local-first
**What**: State lives on disk first. Cloud is optional enhancement.

```yaml
# Config supports both
models:
  providers:
    anthropic:           # Cloud provider
      apiKey: ${ANTHROPIC_API_KEY}
    ollama:              # Local provider
      baseUrl: http://127.0.0.1:11434

agent:
  model:
    primary: anthropic/claude-sonnet-4-5
    fallbacks:
      - ollama/llama3    # Failover to local
```

**Why it matters**: Works on airplane. Privacy. Cost control. Resilience.

---

### App Dissolution
**What**: Skills replace apps. Agent calls tools directly instead of opening GUIs.

**Before** (user workflow):
```
1. Open weather app
2. Search location
3. Read result
4. Open calendar app
5. Create reminder
```

**After** (agent workflow):
```markdown
# SKILL.md for weather
dependencies: [curl]
---
To check weather, run: `curl wttr.in/{location}`
```

Agent just executes: `curl wttr.in/London && openclaw cron add "umbrella reminder"`

**Why it matters**: Removes UI friction. Agent becomes the universal interface.

---

## 2. Gateway Communication

### Hub-and-Spoke
**What**: Central hub (Gateway) with peripheral spokes (clients/nodes).

```typescript
// All spokes register with hub
class Gateway {
  private nodes = new Map<string, WebSocket>();
  private capabilities = new Map<string, string[]>();

  onNodeConnect(ws: WebSocket, nodeId: string, caps: string[]) {
    this.nodes.set(nodeId, ws);
    this.capabilities.set(nodeId, caps);
    // Now gateway knows: "iphone-1 can do camera.snap"
  }

  routeCommand(command: string) {
    // Find node with capability
    for (const [nodeId, caps] of this.capabilities) {
      if (caps.includes(command)) {
        return this.nodes.get(nodeId);
      }
    }
  }
}
```

**Why it matters**: Gateway decides WHERE to execute. Agent just says "take photo" — Gateway routes to iPhone, not Linux server.

---

### WebSocket over REST
**What**: Persistent bidirectional connection instead of request-response.

**REST** (bad for agents):
```
Client: POST /chat {message}
Server: 200 OK {response}  // Must wait for ENTIRE response
```

**WebSocket** (good for agents):
```
Client: → {type: "req", method: "chat.send", message: "..."}
Server: ← {type: "event", event: "thinking", data: "..."}
Server: ← {type: "event", event: "delta", text: "The"}
Server: ← {type: "event", event: "delta", text: " answer"}
Server: ← {type: "event", event: "delta", text: " is..."}
Server: ← {type: "event", event: "tool_call", name: "exec"}
Server: ← {type: "res", ok: true, result: "..."}
```

**Why it matters**: Real-time streaming. See agent thinking. Server can push events (notifications, other user messages).

---

### Node Capability Advertisement
**What**: Nodes tell Gateway what they can do on connect.

```typescript
// iPhone connects and advertises
ws.send(JSON.stringify({
  type: "req",
  method: "node.register",
  params: {
    nodeId: "iphone-14-pro",
    capabilities: [
      "camera.snap",
      "camera.clip",
      "location.get",
      "notification.send",
      "voice.wake"
    ]
  }
}));
```

Gateway maintains routing table:
```
camera.snap    → iphone-14-pro
camera.snap    → macbook-webcam
location.get   → iphone-14-pro
exec           → mac-studio
screen.capture → mac-studio
```

**Why it matters**: Dynamic capability discovery. Plug in new device = instant new capabilities.

---

### TypeBox Validation
**What**: Runtime JSON schema validation using TypeBox library.

```typescript
import { Type, Static } from '@sinclair/typebox';

// Define schema
const ChatMessage = Type.Object({
  role: Type.Union([Type.Literal('user'), Type.Literal('assistant')]),
  content: Type.String(),
  timestamp: Type.Optional(Type.Number())
});

type ChatMessage = Static<typeof ChatMessage>;

// Validate at runtime
import { Value } from '@sinclair/typebox/value';

function handleMessage(raw: unknown) {
  if (!Value.Check(ChatMessage, raw)) {
    const errors = Value.Errors(ChatMessage, raw);
    throw new Error(`Invalid message: ${[...errors]}`);
  }
  // Now TypeScript knows `raw` is ChatMessage
}
```

**Why it matters**: Catches malformed messages before they crash the system. TypeScript types + runtime checks in one definition.

---

## 3. Agent Loop

### Erlang-style Process Isolation
**What**: Agent runs in separate process. Gateway survives agent crashes.

```
┌─────────────────────────────────────────┐
│              GATEWAY PROCESS            │
│  - Always running                       │
│  - Handles WebSocket connections        │
│  - Routes messages                      │
│                                         │
│    ┌─────────────────────────────────┐  │
│    │       AGENT PROCESS (Pi)       │  │
│    │  - Can crash on bad input      │  │
│    │  - Can hang on infinite loop   │  │
│    │  - Can OOM on huge context     │  │
│    └─────────────────────────────────┘  │
│              ↑ restart if dead          │
└─────────────────────────────────────────┘
```

```typescript
// Gateway spawns and monitors agent
class AgentManager {
  async runAgent(message: string) {
    const child = spawn('node', ['pi-runtime.js']);

    child.on('exit', (code) => {
      if (code !== 0) {
        this.log('Agent crashed, but Gateway lives on');
        this.notifyUser('Agent restarting...');
      }
    });

    // Communicate via RPC
    child.stdin.write(JSON.stringify({ message }));
  }
}
```

**Why it matters**: System stays up even when agent goes haywire. Auto-recovery.

---

### Block Streaming
**What**: Stream discrete blocks (thoughts, tool calls, text) not just token deltas.

```typescript
// Events streamed to client
{ type: "block", kind: "thinking", content: "I need to check the file..." }
{ type: "block", kind: "tool_call", name: "read", params: {path: "/foo.txt"} }
{ type: "block", kind: "tool_result", content: "File contents here..." }
{ type: "block", kind: "text", content: "Based on the file, " }
{ type: "block", kind: "text", content: "the answer is 42." }
{ type: "block", kind: "end" }
```

**Why it matters**: UI can render tool calls separately. Show "thinking" in a different style. Progress indicators during long operations.

---

### Thinking Levels
**What**: Adjustable cognitive depth per session.

```yaml
# Session config
sessions:
  coding-project:
    thinkingLevel: high      # Complex reasoning, slower
    model: claude-opus-4-5

  casual-chat:
    thinkingLevel: low       # Quick responses
    model: claude-haiku-4-5
```

Levels: `off` → `low` → `medium` → `high` → `xhigh`

| Level | Behavior |
|-------|----------|
| `off` | Direct response, no reasoning shown |
| `low` | Brief internal check |
| `high` | Extended chain-of-thought |
| `xhigh` | Deep reasoning, multiple passes |

**Why it matters**: Cost/speed tradeoff. Use Haiku for "what time is it?", Opus for "refactor this codebase".

---

### Adaptive Compaction
**What**: Smart context management when approaching token limits.

```typescript
class SessionManager {
  async compact(session: Session) {
    // 1. Trigger hook — let agent save important stuff
    await this.hooks.emit('before_compaction', { session });

    // 2. Summarize old messages
    const summary = await this.summarize(session.messages.slice(0, -10));

    // 3. Replace old messages with summary
    session.messages = [
      { role: 'system', content: `Previous context: ${summary}` },
      ...session.messages.slice(-10)  // Keep recent messages
    ];

    // 4. Notify UI
    this.broadcast('compaction_complete', { sessionId: session.id });
  }
}
```

**Memory flush pattern**:
```markdown
<!-- Agent writes to MEMORY.md before compaction -->
## 2026-01-31: Project Discussion
- User prefers TypeScript over JavaScript
- Working on OpenClaw fork
- Key decision: Use WebSocket not REST
```

**Why it matters**: Long conversations don't crash. Important context preserved. User sees what's happening.

---

## 4. Session & Concurrency

### Session Lanes (Promise Mutex)
**What**: One agent operation per session at a time. Queue the rest.

```typescript
class SessionLane {
  private queue: Promise<void> = Promise.resolve();

  async execute<T>(sessionId: string, fn: () => Promise<T>): Promise<T> {
    // Chain onto existing queue
    const result = this.queue.then(fn);

    // Update queue tail (ignore errors for queue management)
    this.queue = result.then(() => {}, () => {});

    return result;
  }
}

// Usage
const lane = new SessionLane();

// These execute sequentially, not in parallel
lane.execute('main', () => agent.process("What is 2+2?"));
lane.execute('main', () => agent.process("Now multiply by 3"));
// Second message waits for first to complete
```

**Why it matters**: No race conditions. Messages processed in order. No Redis needed.

---

### A2A Communication
**What**: Agents can talk to other agents.

```typescript
// Main agent spawns helper agents
const tools = {
  sessions_send: async (params) => {
    // Send message to another session and wait for response
    const result = await gateway.rpc('sessions.send', {
      targetSession: params.sessionId,
      message: params.message,
      waitForResponse: true
    });
    return result;
  },

  sessions_list: async () => {
    // See what other sessions exist
    return gateway.rpc('sessions.list');
  }
};

// Agent can now coordinate:
// "Send research task to 'researcher' session,
//  send coding task to 'coder' session,
//  aggregate results"
```

**Why it matters**: Parallel work. Specialized agents. Divide and conquer.

---

### Hierarchical Command
**What**: Main agent delegates to sub-agents with restricted permissions.

```
Main Agent (full permissions)
    │
    ├── Sub-agent: "researcher"
    │   └── Tools: [web_search, web_fetch] only
    │
    ├── Sub-agent: "coder"
    │   └── Tools: [read, write, exec] only
    │
    └── Sub-agent: "communicator"
        └── Tools: [telegram, discord] only
```

```typescript
const subagent = await spawnSubagent({
  parentSession: 'main',
  role: 'researcher',
  toolPolicy: {
    allow: ['web_search', 'web_fetch'],
    deny: ['exec', 'write']  // Can't run commands or modify files
  },
  task: 'Research latest AI papers on context compression'
});

const result = await subagent.waitForCompletion();
```

**Why it matters**: Least privilege. Untrusted tasks in sandboxed agents. Parallel execution.

---

## 5. Protocols

### ACP Bridge (IDE Integration)
**What**: Protocol adapter between IDE (VS Code/Zed) and Gateway.

```
┌─────────────┐    stdio     ┌─────────────┐    WebSocket    ┌─────────────┐
│    IDE      │◄────────────►│ ACP Bridge  │◄────────────────►│  Gateway    │
│ (VS Code)   │  JSON-RPC    │ (openclaw   │  Gateway Proto   │             │
│             │              │  acp)       │                  │             │
└─────────────┘              └─────────────┘                  └─────────────┘
```

**Key insight**: IDE sessions are ephemeral (close editor = gone). Gateway sessions are persistent. Bridge maps between them.

```typescript
// Bridge maintains mapping
const sessionMap = {
  'vscode-temp-123': 'agent:main:main',  // Ephemeral → Persistent
  'vscode-temp-456': 'agent:main:main',  // Multiple IDE sessions → same agent
};

// Close VS Code, reopen → still have context!
```

**Why it matters**: IDE integration without losing conversation history.

---

### A2UI Canvas (Declarative UI)
**What**: Agent describes UI intent, system renders natively.

**Agent outputs** (declarative JSON):
```json
{
  "type": "form",
  "title": "Schedule Meeting",
  "fields": [
    {"name": "date", "type": "date-picker"},
    {"name": "attendees", "type": "multi-select", "options": ["Alice", "Bob"]},
    {"name": "room", "type": "dropdown", "options": ["Room A", "Room B"]}
  ],
  "submit": {"action": "schedule_meeting", "session": "main"}
}
```

**Canvas Host renders** to native components:
- Web → React components
- iOS → SwiftUI
- Android → Jetpack Compose

**Why it matters**: Rich UI without agent generating raw HTML (security risk). Cross-platform. Consistent look.

---

### SKILL.md Standard
**What**: Natural language skill definitions that agent learns in-context.

```markdown
---
name: twitter-poster
description: Post tweets on behalf of user
dependencies:
  - bird  # CLI tool
permissions:
  - network
  - cookies
---

# How to Post a Tweet

1. First, ensure `bird` CLI is installed
2. Run: `bird tweet "your message here"`
3. For threads, use: `bird thread "first" "second" "third"`

## Examples

User: "Tweet about the new release"
→ `bird tweet "Just shipped v2.0! Check it out at..."`

User: "Post a thread about AI trends"
→ `bird thread "AI trends for 2026:" "1. Local-first agents" "2. ..."`
```

**Why it matters**: Add capabilities WITHOUT code. Agent reads markdown, learns new skill. Users can write skills.

---

## 6. Security

### TCC Awareness
**What**: Respect macOS permission system, fail explicitly.

```typescript
async function captureScreen() {
  // Check permission first
  const hasPermission = await checkTCCPermission('screen-capture');

  if (!hasPermission) {
    return {
      error: 'PERMISSION_MISSING',
      permission: 'screen-capture',
      instructions: 'Go to System Settings → Privacy → Screen Recording → Enable OpenClaw'
    };
  }

  return await ScreenCaptureKit.capture();
}
```

**Why it matters**: No silent failures. User knows exactly what's missing. Security boundaries respected.

---

### Docker Sandbox
**What**: Run untrusted code in disposable containers.

```typescript
async function runUntrusted(code: string, sessionId: string) {
  const container = await docker.createContainer({
    Image: 'openclaw-sandbox:latest',
    Cmd: ['node', '-e', code],
    HostConfig: {
      NetworkMode: 'none',           // No network
      ReadonlyRootfs: true,          // Can't modify system
      Memory: 512 * 1024 * 1024,     // 512MB limit
      CpuQuota: 50000,               // 50% CPU
      AutoRemove: true               // Delete when done
    }
  });

  await container.start();
  const output = await container.wait();
  // Container auto-deleted, no traces left
  return output;
}
```

**Why it matters**: Discord users can't `rm -rf /`. Code execution without risk.

---

### DM Pairing
**What**: Unknown users must be approved before chatting.

```
Unknown User                     Gateway                         Owner
     │                              │                               │
     │── "Hello" ─────────────────►│                               │
     │                              │── "Pairing request: ABC123" ─►│
     │◄─ "Enter code ABC123" ──────│                               │
     │                              │                               │
     │                              │◄─ "approve ABC123" ───────────│
     │                              │                               │
     │◄─ "You're now connected" ───│                               │
     │                              │                               │
```

**Why it matters**: Prevents prompt injection from strangers. No spam. Explicit trust establishment.

---

### 8-Layer Tool Policy
**What**: Tools filtered through 8 layers before reaching agent.

```
Raw Tools (everything)
        │
        ▼ Profile (minimal/coding/messaging/full)
        │
        ▼ Global (config-wide allow/deny)
        │
        ▼ Provider (Claude vs GPT capabilities)
        │
        ▼ Agent (per-agent overrides)
        │
        ▼ Group (channel/sender restrictions)
        │
        ▼ Sandbox (container isolation)
        │
        ▼ Subagent (spawned agents restricted)
        │
        ▼ Plugin (extension tool gating)
        │
Final Tools (what agent actually sees)
```

Example flow:
```yaml
# Layer 1: Profile = "coding" → allows exec, read, write
# Layer 2: Global deny = ["rm", "sudo"]
# Layer 3: Provider = Claude → allows all
# Layer 4: Agent = "assistant" → no overrides
# Layer 5: Group = Discord #general → deny exec for non-admins
# Layer 6: Sandbox = false → no restrictions
# Layer 7: Subagent = true → deny sessions_*, memory_*
# Layer 8: Plugin = none

# Result: This subagent in Discord can read/write but not exec
```

**Why it matters**: Defense in depth. One layer fails, others protect. Fine-grained control.

---

## 7. Toolchain ("Agent's Organs")

### Peekaboo (Vision)
**What**: Screen capture + accessibility API for GUI automation.

```bash
# See what's on screen (returns JSON of UI elements)
peekaboo see --app "Safari"

# Output:
{
  "windows": [{
    "title": "Google",
    "elements": [
      {"type": "button", "label": "Search", "x": 450, "y": 320},
      {"type": "textfield", "label": "Search input", "x": 400, "y": 280}
    ]
  }]
}

# Click element
peekaboo click --x 450 --y 320

# Type text
peekaboo type "Hello world"
```

**Why it matters**: Agent can control ANY app, even without API. GUI automation for legacy software.

---

### sweet-cookie (Identity)
**What**: Extract browser cookies for session inheritance.

```bash
# Extract Twitter cookies from Chrome
sweet-cookie chrome twitter.com

# Output: cookies ready for use
# Agent can now act as "you" on Twitter
```

Combined with `bird`:
```bash
# Post tweet as yourself (no API key needed)
bird tweet "Hello from my AI agent!"
```

**Why it matters**: Agent inherits your logged-in sessions. No API keys for services that don't have APIs.

---

### Nix Flakes (Reproducibility)
**What**: Package skills with exact dependencies.

```nix
# flake.nix for a skill
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }: {
    packages.x86_64-darwin.default = nixpkgs.legacyPackages.x86_64-darwin.buildEnv {
      name = "media-skill";
      paths = [
        nixpkgs.legacyPackages.x86_64-darwin.ffmpeg
        nixpkgs.legacyPackages.x86_64-darwin.imagemagick
        nixpkgs.legacyPackages.x86_64-darwin.yt-dlp
      ];
    };
  };
}
```

```bash
# Run skill in isolated environment
nix run .#media-skill -- ffmpeg -i input.mp4 output.gif
```

**Why it matters**: Skills work identically everywhere. No "works on my machine". Clean uninstall.

---

## Summary Cheatsheet

| Pattern | One-liner |
|---------|-----------|
| Sovereign AI | Your data on your disk |
| OS-as-Surface | Shell is the API |
| Gateway-first | Single WebSocket hub |
| Local-first | Disk before cloud |
| Hub-and-Spoke | Central routing table |
| TypeBox | Runtime + static types |
| Process Isolation | Agent crashes, Gateway lives |
| Block Streaming | Stream thoughts, not just tokens |
| Thinking Levels | Haiku for chat, Opus for code |
| Adaptive Compaction | Summarize before overflow |
| Session Lanes | Promise queue per session |
| A2A | Agents talk to agents |
| ACP Bridge | IDE ↔ Gateway translator |
| A2UI | Declarative UI from agent |
| SKILL.md | Natural language capabilities |
| TCC Awareness | Explicit permission errors |
| Docker Sandbox | Disposable execution |
| DM Pairing | Approve before chat |
| 8-Layer Policy | Defense in depth |
| Peekaboo | See and click GUIs |
| sweet-cookie | Inherit browser sessions |
| Nix Flakes | Reproducible dependencies |

---

*Written by Claude (Opus 4.5) | 2026-01-31 PST*
