# Key Architecture Patterns from OpenClaw

Notes extracted from lencx's WeChat article "深度解析：Moltbot 底层架构" (2026-01-29).

> **Note**: The project was renamed from "clawdbot" to "moltbot" to "openclaw".

---

## 1. Design Philosophy Patterns

| Pattern | Description |
|---------|-------------|
| **Sovereign AI** | Data stays local; user owns all data physically (Markdown files, not cloud DBs) |
| **OS-as-Surface** | OS itself is the AI interface; CLI > GUI for agent interaction (LLMs understand shell natively) |
| **Gateway-first** | Single control plane unifies all channels/clients/nodes via observable WS + events |
| **Local-first** | Sessions/state stored locally; supports both cloud and local providers (Ollama) with failover |
| **App Dissolution** | Skills (SKILL.md) replace apps; agent calls `curl`/`ffmpeg` directly instead of opening apps |

### Why CLI over GUI/MCP?

LLMs are pre-trained on massive amounts of shell scripts and system docs. They natively understand how to combine `grep`, `awk`, `curl` to solve problems. Unix-style CLI is the agent's most natural language.

---

## 2. Gateway Communication Patterns

| Pattern | Description |
|---------|-------------|
| **Hub-and-Spoke** | Gateway = single source of truth; all clients (WhatsApp/iOS/CLI) connect as spokes |
| **WebSocket over REST** | Full-duplex WS for real-time agent streaming; abandons RESTful for control plane |
| **Node Capability Advertisement** | Nodes broadcast capabilities (`["camera.snap", "location.get"]`); Gateway maintains dynamic routing table |
| **TypeBox Schema Validation** | All WS messages validated against TypeBox schemas; eliminates runtime type errors |
| **Tailscale Integration** | Remote access via WireGuard tunnel without opening firewall ports |

### Architecture Overview

```
Chat Channels (WhatsApp/Telegram/Discord/Slack/...)
                    |
                    v
            Moltbot Gateway (daemon)
            - owns channel connections
            - WS control plane + HTTP surfaces
            - agent loop execution + queues + session storage
            - tools router (browser/nodes/exec/cron/...)
                    |
    +---------------+---------------+
    v               v               v
Operator Clients   Nodes          Plugins/Skills
(CLI/Web UI/       (iOS/Android/  (channels/tools/ui
 macOS app)        macOS/headless) schema/hooks)
```

### WebSocket Protocol Flow

```
Client                          Gateway
|------ req:connect ----------->|
|<----- res:hello-ok -----------|
|<----- event:tick -------------|
|------ req:health ------------>|
|<----- res:health -------------|
```

---

## 3. Agent Loop Patterns

| Pattern | Description |
|---------|-------------|
| **Process Isolation (Erlang-style)** | Pi Runtime as separate process; Gateway survives agent crashes and restarts it |
| **Block Streaming** | Thoughts/tool-calls/responses stream as independent blocks in real-time |
| **Thinking Levels** | Dynamic cognitive depth (`off` → `xhigh`); routes to different models per level |
| **Adaptive Compaction** | Recursive summarization + `before_compaction` hook to flush important info to long-term storage before context truncation |
| **Voice Loop with VAD** | Local Voice Activity Detection for natural turn-taking; low-power wake word detection |

### Erlang-style Fault Tolerance

The core philosophy: keep the scheduling layer alive; put risky computation in isolated processes. If a child process crashes, the core can restart it. This is supervisor-style fault tolerance borrowed from Erlang/OTP.

### Adaptive Compaction Safeguard

1. **Dynamic Chunking** — Auto-split large tasks into atomic units when approaching context limit
2. **Recursive Summarization** — Summarize history, keep key decisions, discard noise
3. **Memory Flush** — `before_compaction` hook prompts model to write important info to `memory/` before compression
4. **UI Feedback** — Compression status pushed to UI in real-time

---

## 4. Session & Concurrency Patterns

| Pattern | Description |
|---------|-------------|
| **Session Lanes** | Promise-based mutex per session; prevents race conditions without Redis |
| **Request Queuing** | New messages queue while agent processes previous; no state corruption |
| **A2A Communication** | `sessions_list`, `sessions_history`, `sessions_send` enable agent-to-agent collaboration |
| **Hierarchical Command** | Main agent spawns sub-agents with restricted permissions; results aggregate back |
| **State Persistence** | All session config (thinkingLevel, model, policy) persisted to `sessions.json` |

### Agent-to-Agent (A2A) Collaboration

- `sessions_list` — Query active sessions and their metadata
- `sessions_history` — Read other session's history for knowledge sharing
- `sessions_send` — Send message to another agent (supports ping-pong mode)

**Use case**: Main session agent acts as "commander", spawns sub-sessions for parallel work (e.g., "Twitter analyst", "news aggregator", "data cleaner"), then aggregates results.

---

## 5. Protocol & Extension Patterns

| Pattern | Description |
|---------|-------------|
| **ACP Bridge** | Protocol translation: stdio JSON-RPC ↔ Gateway WS protocol; IDE sessions map to persistent Gateway sessions |
| **A2UI Canvas** | Agent generates declarative UI JSON; Canvas Host renders to native components (React/SwiftUI/etc.) |
| **SKILL.md Standard** | Natural language skill definitions (YAML frontmatter + markdown body); in-context learning |
| **Nix Flakes** | Reproducible, isolated dependency environments for skills; no host pollution |

### A2UI Architecture

```
+------------------+     +-------------------+     +--------------+
| Canvas Host      |---->| Node Bridge       |---->| Node App     |
| (HTTP Server)    |     | (TCP Server)      |     | (Mac/iOS/    |
| Port 18793       |     | Port 18790        |     | Android)     |
+------------------+     +-------------------+     +--------------+
```

- Agent generates **declarative UI JSON** (not raw HTML)
- Canvas Host renders to native components
- Enables "micro-apps" within conversation (voting UI, dashboards, etc.)

---

## 6. Security Patterns

| Pattern | Description |
|---------|-------------|
| **TCC Awareness** | Gateway returns `PERMISSION_MISSING` instead of silent failure when macOS permissions lacking |
| **Docker Sandbox** | Untrusted code runs in ephemeral containers; destroyed after session |
| **DM Pairing** | Unknown DMs blocked; require out-of-band approval code to interact |
| **TLS Pinning** | Mobile nodes verify Gateway certificate fingerprint; prevents MitM |
| **8-Layer Tool Policy** | Profile → Global → Provider → Agent → Group → Sandbox → Subagent → Plugin filtering |

### Authentication Flow

1. Client connects to `ws://<gateway-ip>:18789`
2. Must provide pre-generated auth token (from "Hatching" phase)
3. Mobile devices use pairing flow (short code + admin approval)
4. TLS Pinning for Tailscale tunnels

---

## 7. Toolchain Integration Pattern

**"Agent's Organs"** — External CLI tools become agent capabilities:

| Tool | Purpose |
|------|---------|
| `Peekaboo` | Screen capture + Accessibility API for GUI automation |
| `sweet-cookie` | Browser cookie extraction for session inheritance |
| `bird` | Twitter/X client using extracted cookies |
| `imsg` | iMessage read/send |
| `summarize` | URL/PDF/YouTube summarization |
| `oracle` | Web search |
| `Poltergeist` | Click, type, control macOS UI |
| `sag` | Text-to-speech (TTS) |
| `camsnap` | Camera capture |
| `go-cli` | Google Calendar integration |
| `sonoscli` | Sonos speaker control |

### Peekaboo: Machine Vision & Accessibility

- **Hybrid enumeration**: ScreenCaptureKit for capture + CGWindowList for window coords
- **Accessibility API**: `peekaboo see` returns pruned JSON of UI elements with coordinates
- **Operation loop**: Agent parses JSON, issues `peekaboo click` or `peekaboo type`
- Extends agent capability to entire GUI desktop

---

## Key Insight

> The architecture isn't just a chatbot—it's an **AI-native OS layer**. Gateway unifies the control plane, OS-as-Surface connects the system layer, A2UI/ACP connect upper applications. Together they define a complete "Sovereign AI" infrastructure standard.

---

## References

- [Moltbot/OpenClaw](https://github.com/moltbot/moltbot)
- [TypeBox](https://github.com/sinclairzx81/typebox)
- [Tailscale](https://tailscale.com)
- [Pi Agent Core](https://github.com/badlogic/pi-mono)
- [ACP (Agent Client Protocol)](https://github.com/agentclientprotocol)
- [A2UI](https://github.com/google/A2UI)
- [Peekaboo](https://github.com/steipete/Peekaboo)
- [sweet-cookie](https://github.com/steipete/sweet-cookie)
- [nix-moltbot](https://github.com/moltbot/nix-moltbot)

---

*Source: lencx (浮之静) WeChat article "深度解析：Moltbot 底层架构" | 2026-01-29*
*Notes by Claude (Opus 4.5) | 2026-01-31 PST*
