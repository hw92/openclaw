# Moltbot Learning Notes

Personal notes from studying the moltbot codebase.

> **Note**: The project was renamed from "clawdbot" to "moltbot" in January 2026. These notes have been updated to reflect the new name.

## Documents

| Date | Document | Description |
|------|----------|-------------|
| 2026-01-26 | [Fork Setup](2026-01-26-fork-setup-moltbot.md) | Setting up a tracking fork |
| 2026-01-26 | [Architecture Reference](2026-01-26-moltbot-architecture.md) | Comprehensive codebase overview |
| 2026-01-26 | [Architecture Deep Dive](2026-01-26-architecture-deep-dive.md) | Conceptual insights and deeper understanding |
| 2026-01-31 | [Architecture Patterns (WeChat)](2026-01-31-architecture-patterns-from-wechat-analysis.md) | Key patterns from lencx's Chinese analysis |
| 2026-01-31 | [Patterns Crash Course](2026-01-31-architecture-patterns-crash-course.md) | Practical deep-dive with code examples |
| 2026-01-31 | [Patterns in Codebase](2026-01-31-patterns-in-codebase.md) | Mapping patterns to actual source files |

## Topics Covered

- **Overview** — What moltbot is, hub-and-spoke architecture
- **Agent System** — Pi agent runtime, bootstrap files, tools, sessions
- **Messaging Channels** — Channel plugins, routing, adapters, chunking
- **Gateway** — WebSocket server, RPC methods, lanes, authentication
- **Plugin System** — Plugin SDK, lifecycle hooks, discovery
- **Providers** — LLM providers, auth profiles, failover
- **Companion Apps** — macOS, iOS, Android apps
- **Design Patterns** — Sovereign AI, OS-as-Surface, Gateway-first, Local-first
- **Concurrency** — Session lanes, A2A collaboration, request queuing
- **Security** — TCC awareness, Docker sandbox, DM pairing, 8-layer tool policy
- **Toolchain** — Peekaboo, sweet-cookie, bird, SKILL.md standard, Nix Flakes
