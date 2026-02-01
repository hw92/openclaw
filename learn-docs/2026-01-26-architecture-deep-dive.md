# Moltbot Architecture Deep Dive

Deeper architectural insights and conceptual understanding beyond the reference documentation.

> **Note**: The project was renamed from "clawdbot" to "moltbot" in January 2026.

---

## Table of Contents

1. [Agent as the AI-Native Core](#1-agent-as-the-ai-native-core)
2. [How the Agent Uses Tools](#2-how-the-agent-uses-tools)

---

## 1. Agent as the AI-Native Core

The agent (`src/agents/`) is the **AI-native, non-deterministic core** of moltbot, while everything else is **deterministic infrastructure**.

### The Split

```
┌─────────────────────────────────────────────────────────────┐
│                    DETERMINISTIC INFRASTRUCTURE              │
│  (predictable, same input → same output)                    │
├─────────────────────────────────────────────────────────────┤
│  Gateway      │  Channels     │  Plugins    │  Providers    │
│  - Routing    │  - Adapters   │  - Loading  │  - API calls  │
│  - Auth       │  - Chunking   │  - Hooks    │  - Failover   │
│  - WebSocket  │  - Delivery   │  - Registry │  - Auth       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    AI-NATIVE / NON-DETERMINISTIC            │
│  (uncertain, creative, reasoning)                           │
├─────────────────────────────────────────────────────────────┤
│                         AGENT                               │
│  - LLM reasoning loop                                       │
│  - Tool selection (which tool? what params?)               │
│  - Response generation                                      │
│  - Context interpretation                                   │
│  - Multi-turn conversation state                           │
└─────────────────────────────────────────────────────────────┘
```

### Deterministic vs Non-Deterministic

| Aspect | Infrastructure | Agent |
|--------|---------------|-------|
| **Behavior** | Deterministic | Non-deterministic |
| **Output** | Predictable | Creative/variable |
| **Decision making** | Rules/config | Reasoning |
| **Uncertainty** | None (bugs aside) | Intentional |
| **Intelligence** | None | Yes |

### What the Agent Uniquely Provides

The agent is where:

1. **Interpretation happens** — "What does the user actually want?"
2. **Decisions are made** — "Should I use `exec` or `web_search`?"
3. **Creativity emerges** — Responses aren't templated
4. **Context matters** — Same input can yield different outputs based on history
5. **Errors are fuzzy** — Not crashes, but "wrong" reasoning

### Infrastructure = Plumbing

Everything else is **plumbing** to:
- Get messages **to** the agent (channels → gateway → agent)
- Execute agent **decisions** (tools, sends)
- Get responses **from** the agent (agent → gateway → channels)
- Manage **credentials** for the LLM (providers, auth profiles)

The infrastructure doesn't "think" — it just routes, transforms, and executes.

### Why This Separation Matters

| Concern | Infrastructure | Agent |
|---------|---------------|-------|
| **Testing** | Unit tests, deterministic | Evals, fuzzy testing |
| **Debugging** | Reproducible bugs | Prompt engineering |
| **Scaling** | Predictable compute | Token-based costs |
| **Security** | Clear boundaries | Guardrails, sandboxing, tool policies |

### Key Insight

> The agent is the "soul" of the app — everything else exists to serve it.

The entire infrastructure stack (gateway, channels, plugins, providers) is built to:
1. Bring user intent to the agent
2. Let the agent reason and decide
3. Execute the agent's decisions
4. Deliver the agent's response

Without the agent, moltbot would just be a message router. The agent is what makes it an **AI assistant**.

---

## 2. How the Agent Uses Tools

Tools are how the agent **acts on the world**. The LLM reasons and decides; tools execute.

### The Tool Loop

```
┌─────────────────────────────────────────────────────────────┐
│                     PI AGENT LOOP                           │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        ▼                                     │
┌───────────────┐                             │
│  Build Prompt │                             │
│  system +     │                             │
│  history +    │                             │
│  tool schemas │                             │
└───────┬───────┘                             │
        │                                     │
        ▼                                     │
┌───────────────┐                             │
│   Call LLM    │ ◄─────────────────────────┐ │
└───────┬───────┘                           │ │
        │                                   │ │
        ▼                                   │ │
┌───────────────┐     ┌───────────────┐     │ │
│ Parse Response│────►│  tool_use?    │─Yes─┤ │
└───────────────┘     └───────┬───────┘     │ │
                              │ No          │ │
                              ▼             │ │
                      ┌───────────────┐     │ │
                      │ Final Text    │     │ │
                      │ (end turn)    │     │ │
                      └───────────────┘     │ │
                                            │ │
        ┌───────────────────────────────────┘ │
        ▼                                     │
┌───────────────┐                             │
│ Execute Tools │                             │
│ tool.execute()│                             │
└───────┬───────┘                             │
        │                                     │
        ▼                                     │
┌───────────────┐                             │
│ Add Results   │                             │
│ to History    │─────────────────────────────┘
└───────────────┘
```

### Key Insight: Tools as the Agent's Hands

The LLM is a **reasoning engine** that outputs two things:
1. **Text** — Communication to the user
2. **Tool calls** — Actions to perform

Tools are the **bridge between reasoning and reality**:

| LLM Output | What It Is | Example |
|------------|------------|---------|
| Text | Thoughts, responses | "I'll check that file for you" |
| `tool_use` | Action request | `{ "name": "read", "params": { "path": "/foo.txt" } }` |

The agent loop keeps iterating until the LLM decides it's done (returns text without tool calls).

### Tool Definition Anatomy

Every tool has three parts:

```typescript
{
  name: "exec",                    // Canonical name
  schema: Type.Object({            // TypeBox schema (becomes JSON Schema)
    command: Type.String(),
    workdir: Type.Optional(Type.String()),
    timeout: Type.Optional(Type.Number()),
  }),
  execute: async (callId, params) => {   // Implementation
    const result = await runCommand(params.command);
    return { content: [{ type: "text", text: result }] };
  }
}
```

The **schema** is what the LLM sees (in system prompt). The **execute** is what runs when called.

### Tool Categories

```
┌─────────────────────────────────────────────────────────────┐
│                    TOOL TAXONOMY                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FILE SYSTEM          EXECUTION         COMMUNICATION       │
│  ───────────          ─────────         ─────────────       │
│  read                 exec              message             │
│  write                process           sessions_send       │
│  edit                                   telegram            │
│  apply_patch                            discord             │
│                                         slack               │
│                                                             │
│  WEB/BROWSER          SYSTEM            SCHEDULING          │
│  ───────────          ──────            ──────────          │
│  web_fetch            gateway           cron                │
│  web_search           session_status                        │
│  browser              memory_*                              │
│                       agents_list                           │
│                                                             │
│  MEDIA                NODES             PLUGINS             │
│  ─────                ─────             ───────             │
│  image                canvas            (extensible)        │
│  voice                nodes_tool                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The 8-Layer Policy Filter

Tools pass through **8 layers of filtering** before reaching the agent:

```
Raw Tools (all available)
    │
    ▼ Profile policy (minimal/coding/messaging/full)
    │
    ▼ Global policy (config-wide allow/deny)
    │
    ▼ Provider policy (Claude vs GPT vs Gemini)
    │
    ▼ Agent policy (per-agent overrides)
    │
    ▼ Group policy (channel/sender restrictions)
    │
    ▼ Sandbox policy (container isolation)
    │
    ▼ Subagent policy (spawned agents get fewer tools)
    │
    ▼ Plugin policy (extension tool gating)
    │
Filtered Tools (what agent actually sees)
```

**Why so many layers?**
- Security: Don't let untrusted groups run `exec`
- Isolation: Subagents shouldn't manage sessions
- Flexibility: Different models may need different tools

### Tool Context via Closure

Tools receive context through **closure capture** at creation time:

```typescript
function createMessageTool(options: {
  config: MoltbotConfig,
  agentSessionKey: string,
  currentChannelId: string,
}) {
  // Context captured here ▲

  return {
    name: "message",
    execute: async (callId, params) => {
      // Context available here via closure
      const channel = params.channel || options.currentChannelId;
      // ...
    }
  };
}
```

This is why tools are created **per-agent-run**, not globally — each run has different context.

### Tool Result Flow

```
Tool executes
    │
    ▼
Returns AgentToolResult
    │
    ├─► content: [{ type: "text", text: "..." }]  ─► Goes to LLM
    │
    └─► details: { ... }  ─► Metadata for logging/UI
    │
    ▼
Emits tool_execution_end event
    │
    ├─► UI shows tool completion
    ├─► Streaming callbacks fire
    └─► Result added to conversation history
    │
    ▼
LLM sees result in next turn
```

### Error Handling Philosophy

Tools don't crash the agent — they return errors as results:

```typescript
// Tool error becomes LLM input
return {
  content: [{ type: "text", text: "Error: file not found" }]
};

// LLM can then:
// 1. Try a different approach
// 2. Ask the user for clarification
// 3. Report the error gracefully
```

The LLM is **resilient to tool failures** because errors are just text it can reason about.

### Exec Tool: The Special Case

The `exec` tool deserves special attention because it's the most **powerful and dangerous**:

```
User message: "Delete all .tmp files"
    │
    ▼
LLM reasons: "I should use exec to run rm"
    │
    ▼
Tool call: exec({ command: "rm *.tmp" })
    │
    ▼
┌─────────────────────────────────────┐
│         APPROVAL CHECK              │
│  Does this command need approval?   │
│  - Destructive? (rm, mv, etc.)      │
│  - Sensitive? (sudo, etc.)          │
│  - Config says require approval?    │
└─────────────────┬───────────────────┘
                  │
        ┌─────────┴─────────┐
        │ Yes               │ No
        ▼                   ▼
┌───────────────┐   ┌───────────────┐
│ Request User  │   │ Execute       │
│ Approval      │   │ Directly      │
└───────┬───────┘   └───────────────┘
        │
        ▼
┌───────────────┐
│ User approves │─No─► Return "approval denied"
└───────┬───────┘
        │ Yes
        ▼
    Execute command
```

### Key Insight: Tools as Capability Boundaries

Tools define **what the agent can do**. Policy filtering defines **what it's allowed to do**.

| Without Tool | With Tool |
|--------------|-----------|
| Can't read files | Has `read` tool |
| Can't run commands | Has `exec` tool |
| Can't send messages | Has `message` tool |
| Can't browse web | Has `browser` tool |

The agent's power comes entirely from its tools. An agent with no tools is just a chatbot.

### Streaming: Real-time Feedback

Long-running tools emit **progress events**:

```
tool_execution_start  ─► "Starting browser..."
    │
tool_execution_update ─► "Loading page..."
    │
tool_execution_update ─► "Taking screenshot..."
    │
tool_execution_end    ─► "Done: screenshot.png"
```

This lets the UI show what's happening without waiting for completion.

### Plugin Tools

Plugins can register their own tools:

```typescript
// In a plugin
api.registerTool({
  name: "my_custom_tool",
  schema: Type.Object({ ... }),
  execute: async (callId, params) => { ... }
});
```

These get merged into the tool array and filtered by the same policy system.

---

## Summary: The Tool Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│                        AGENT                                │
│                                                             │
│   ┌─────────────┐        ┌─────────────┐                   │
│   │    LLM      │◄──────►│   TOOLS     │                   │
│   │  (reasoning)│        │  (actions)  │                   │
│   └─────────────┘        └─────────────┘                   │
│         │                      │                           │
│         │ decides              │ executes                  │
│         ▼                      ▼                           │
│   "I should read    ───►  read("/foo.txt")                 │
│    that file"              │                               │
│         ▲                  │                               │
│   "The file says    ◄───  "contents..."                    │
│    X, so I'll..."                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The LLM **thinks**. Tools **act**. The loop continues until the LLM is satisfied.

---

*Written by Claude (Opus 4.5) | 2026-01-26 14:45 PST*
*Updated 2026-01-27: Renamed from clawdbot to moltbot*
