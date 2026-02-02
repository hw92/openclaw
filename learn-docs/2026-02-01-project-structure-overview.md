# OpenClaw Project Structure Overview

A beginner-friendly guide to understanding the OpenClaw codebase structure, designed for non-professional developers and vibe coders.

**Last updated:** 2026-02-01

---

## Table of Contents

1. [What is OpenClaw?](#what-is-openclaw)
2. [The Big Picture](#the-big-picture)
3. [Architecture Diagram](#architecture-diagram)
4. [Root Directory Organization](#root-directory-organization)
5. [Core Application (src/)](#core-application-src)
6. [Apps vs Extensions - The Key Difference](#apps-vs-extensions---the-key-difference)
7. [Configuration Files Explained](#configuration-files-explained)
8. [Deployment Options](#deployment-options)
9. [For "Install from Source" - What You Need](#for-install-from-source---what-you-need)
10. [Glossary](#glossary)

---

## What is OpenClaw?

OpenClaw is a **personal AI assistant platform** that:

- Connects to messaging apps (WhatsApp, Telegram, Discord, Slack, iMessage, etc.)
- Runs AI agents (Claude, GPT, Gemini, local models) to respond to your messages
- Provides multiple interfaces: CLI, web dashboard, and native apps (macOS, iOS, Android)
- Can be self-hosted on your own machine or deployed to the cloud

Think of it as your own AI butler that lives across all your messaging platforms.

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER INTERFACES                                │
│                    (how YOU interact with OpenClaw)                      │
│                                                                          │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│   │ macOS    │  │ iOS      │  │ Android  │  │ Web UI   │  │ CLI      │ │
│   │ App      │  │ App      │  │ App      │  │ (ui/)    │  │ Terminal │ │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│        │             │             │             │             │        │
│        └─────────────┴─────────────┴─────────────┴─────────────┘        │
│                                    │                                     │
│                              WebSocket/HTTP                              │
│                                    ▼                                     │
└────────────────────────────────────┼─────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼─────────────────────────────────────┐
│                         OPENCLAW GATEWAY                                 │
│                          (src/gateway/)                                  │
│                                                                          │
│   The "brain" that coordinates everything:                               │
│   - Receives messages from all sources                                   │
│   - Routes to AI providers                                               │
│   - Manages sessions and context                                         │
│   - Sends responses back                                                 │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                      EXTENSIONS (plugins)                        │   │
│   │         Gateway loads these to connect to external services      │   │
│   │                                                                  │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│   │  │ Telegram │ │ WhatsApp │ │ Discord  │ │ Slack    │  ...      │   │
│   │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘           │   │
│   └───────┼────────────┼────────────┼────────────┼──────────────────┘   │
│           │            │            │            │                       │
└───────────┼────────────┼────────────┼────────────┼───────────────────────┘
            │            │            │            │
            ▼            ▼            ▼            ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ Telegram │ │ WhatsApp │ │ Discord  │ │ Slack    │
      │ Servers  │ │ Servers  │ │ Servers  │ │ Servers  │
      └──────────┘ └──────────┘ └──────────┘ └──────────┘
       (external)   (external)   (external)   (external)

                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          AI PROVIDERS                                    │
│              (Claude, GPT, Gemini, local models, etc.)                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Diagram

### Apps vs Extensions - Visual Comparison

```
                    APPS (apps/)                     EXTENSIONS (extensions/)
                    ════════════                     ════════════════════════

                    User-facing clients              Gateway plugins
                    that connect TO gateway          that connect FROM gateway
                                                     to external services

    ┌─────────┐     ┌─────────┐                     ┌─────────┐     ┌─────────┐
    │  You    │────▶│ macOS   │                     │ Gateway │────▶│Telegram │
    │ (user)  │     │  App    │──┐                  │         │     │Extension│──▶ Telegram
    └─────────┘     └─────────┘  │                  │         │     └─────────┘    Servers
                                 │                  │         │
    ┌─────────┐     ┌─────────┐  │   ┌─────────┐   │         │     ┌─────────┐
    │  You    │────▶│ iOS     │──┼──▶│ Gateway │◀──┤         │────▶│WhatsApp │──▶ WhatsApp
    │ (user)  │     │  App    │  │   └─────────┘   │         │     │Extension│    Servers
    └─────────┘     └─────────┘  │                  │         │     └─────────┘
                                 │                  │         │
    ┌─────────┐     ┌─────────┐  │                  │         │     ┌─────────┐
    │  You    │────▶│ Android │──┘                  │         │────▶│ Discord │──▶ Discord
    │ (user)  │     │  App    │                     │         │     │Extension│    Servers
    └─────────┘     └─────────┘                     └─────────┘     └─────────┘

                    Written in:                      Written in:
                    - Swift (iOS/macOS)              - TypeScript
                    - Kotlin (Android)
                                                     Runs inside gateway process
                    Runs on your device
```

---

## Root Directory Organization

The root directory is organized by **purpose**. Here's everything explained:

### Core Application (The Brain)

```
openclaw/
├── src/              ← ALL main TypeScript logic lives here
├── dist/             ← Compiled JavaScript output (after pnpm build)
└── openclaw.mjs      ← CLI entry point (runs dist/entry.js)
```

### Plugins & Extensions

```
├── extensions/       ← Channel and feature plugins
│   ├── telegram/     ← Telegram messaging
│   ├── discord/      ← Discord messaging
│   ├── slack/        ← Slack messaging
│   ├── whatsapp/     ← WhatsApp (if separate from core)
│   ├── matrix/       ← Matrix protocol
│   ├── msteams/      ← Microsoft Teams
│   ├── imessage/     ← Apple iMessage
│   ├── voice-call/   ← Voice call handling
│   ├── memory-lancedb/  ← Vector database for AI memory
│   └── ...           ← 30+ more plugins
```

### Native Applications

```
├── apps/
│   ├── macos/        ← macOS menubar app (Swift)
│   ├── ios/          ← iPhone/iPad app (Swift)
│   ├── android/      ← Android app (Kotlin)
│   └── shared/       ← Shared code between apps
│
└── Swabble/          ← Shared Swift library for iOS/macOS
    ├── Sources/      ← Swift source code
    ├── Tests/        ← Swift tests
    └── Package.swift ← Swift package manifest
```

### Web Interface

```
└── ui/               ← Web dashboard (React/Vite)
    ├── src/          ← React components
    ├── package.json  ← UI dependencies
    └── ...           ← Vite config, etc.
```

### Development Tools

```
├── scripts/          ← Build, release, test helper scripts
│   ├── build-and-run-mac.sh
│   ├── package-mac-app.sh
│   └── ...
│
├── test/             ← Test utilities and fixtures
├── packages/         ← Internal shared TypeScript libraries
├── vendor/           ← Third-party code included directly
│   └── a2ui/         ← UI library for canvas feature
│
├── patches/          ← Fixes for npm dependencies
└── git-hooks/        ← Git hook scripts
```

### Documentation

```
├── docs/             ← User documentation (docs.openclaw.ai)
├── learn-docs/       ← Learning materials (like this file!)
├── skills/           ← AI agent skill definitions (markdown prompts)
│   ├── code-review.md
│   ├── commit.md
│   └── ...           ← Pre-written prompts for the AI
│
├── README.md         ← Project overview (shown on GitHub)
├── CONTRIBUTING.md   ← How to contribute
├── CHANGELOG.md      ← Version history
├── SECURITY.md       ← Security policy
└── docs.acp.md       ← ACP Bridge documentation
```

### Deployment Configurations

```
├── Dockerfile            ← Main Docker image
├── Dockerfile.sandbox    ← Sandboxed execution environment
├── Dockerfile.sandbox-browser  ← Browser sandbox
├── docker-compose.yml    ← Local Docker setup
├── docker-setup.sh       ← Docker initialization script
│
├── fly.toml              ← Fly.io deployment (public)
├── fly.private.toml      ← Fly.io deployment (private settings)
└── render.yaml           ← Render.com deployment
```

---

## Core Application (src/)

The `src/` folder contains all the main application logic:

```
src/
├── index.ts          ← Main exports
├── entry.ts          ← CLI entry point
│
├── cli/              ← Command-line interface
│   ├── program/      ← CLI command definitions
│   │   ├── build-program.ts
│   │   └── command-registry.ts  ← All commands registered here
│   └── deps.ts       ← Dependency injection
│
├── commands/         ← Individual CLI command implementations
│   ├── gateway.ts    ← `openclaw gateway` command
│   ├── agent.ts      ← `openclaw agent` command
│   ├── status.ts     ← `openclaw status` command
│   └── ...
│
├── gateway/          ← WebSocket server (the brain)
├── channels/         ← Built-in channel handlers
├── agents/           ← AI agent runtime
├── providers/        ← LLM provider integrations (Claude, GPT, etc.)
├── config/           ← Configuration schema and validation
├── media/            ← Media processing (images, audio, video)
├── plugins/          ← Plugin loading system
├── terminal/         ← Terminal UI and output formatting
└── ...
```

---

## Apps vs Extensions - The Key Difference

This is a common point of confusion, so let's be very clear:

### Apps (`apps/` folder)

| Aspect | Description |
|--------|-------------|
| **What** | Native applications for your devices |
| **Role** | User interface - how YOU interact with OpenClaw |
| **Direction** | Connect TO the gateway |
| **Language** | Swift (iOS/macOS), Kotlin (Android) |
| **Runs on** | Your phone, tablet, or computer |
| **Examples** | macOS menubar app, iOS app, Android app |

### Extensions (`extensions/` folder)

| Aspect | Description |
|--------|-------------|
| **What** | Plugins loaded by the gateway |
| **Role** | Service connectors - how gateway reaches external services |
| **Direction** | Connect FROM gateway to external services |
| **Language** | TypeScript |
| **Runs on** | Inside the gateway process |
| **Examples** | Telegram, Discord, Slack, WhatsApp plugins |

### Example Message Flows

**Flow 1: Someone messages you on Telegram**

```
Telegram User sends message
         │
         ▼
Telegram Servers
         │
         ▼
┌────────────────────┐
│ Telegram Extension │ ← Receives from Telegram API
│   (in Gateway)     │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│     Gateway        │ ← Routes to AI
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│   AI Provider      │ ← Generates response
│   (e.g., Claude)   │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│     Gateway        │ ← Receives AI response
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Telegram Extension │ ← Sends via Telegram API
└────────┬───────────┘
         │
         ▼
Telegram Servers → Telegram User receives reply
```

**Flow 2: You use the macOS app**

```
You type message in macOS app
         │
         ▼
┌────────────────────┐
│     macOS App      │ ← User interface
└────────┬───────────┘
         │ WebSocket
         ▼
┌────────────────────┐
│     Gateway        │ ← Routes to AI
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│   AI Provider      │ ← Generates response
│   (e.g., Claude)   │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│     Gateway        │ ← Sends response back
└────────┬───────────┘
         │ WebSocket
         ▼
┌────────────────────┐
│     macOS App      │ ← Displays response
└────────────────────┘
         │
         ▼
You see the response
```

---

## Configuration Files Explained

### Package Management Files

| File | Purpose |
|------|---------|
| `package.json` | Main package manifest (name, version, dependencies, scripts) |
| `pnpm-lock.yaml` | Exact dependency versions for reproducible builds |
| `pnpm-workspace.yaml` | Defines monorepo structure (which folders are packages) |
| `.npmrc` | npm/pnpm security settings (whitelist for build scripts) |

### TypeScript Configuration

| File | Purpose |
|------|---------|
| `tsconfig.json` | TypeScript compiler settings (target, module format, strict mode) |

### Code Quality Tools

| File | Purpose |
|------|---------|
| `.oxlintrc.json` | Linter configuration (code style rules) |
| `.oxfmtrc.jsonc` | Formatter configuration (code prettification) |
| `.swiftlint.yml` | Swift linter for iOS/macOS code |
| `.swiftformat` | Swift formatter settings |
| `.shellcheckrc` | Shell script linter settings |
| `.pre-commit-config.yaml` | Checks to run before each git commit |

### Hidden Dot Folders

| Folder | Purpose |
|--------|---------|
| `.git/` | Git version control data |
| `.github/` | GitHub Actions (CI/CD automation) |
| `.pi/` | Pi Agent configuration (AI coding agent settings) |
| `.claude/` | Claude Code workspace settings |
| `.agent/` | General agent-related configs |

### Test Configuration Files

| File | Purpose |
|------|---------|
| `vitest.config.ts` | Main test configuration |
| `vitest.unit.config.ts` | Unit tests only |
| `vitest.e2e.config.ts` | End-to-end tests |
| `vitest.live.config.ts` | Tests with real API keys |
| `vitest.gateway.config.ts` | Gateway-specific tests |
| `vitest.extensions.config.ts` | Plugin tests |

### AI Agent Files

| File | Purpose |
|------|---------|
| `AGENTS.md` | Instructions for AI coding agents (Claude, Cursor, Copilot) |
| `CLAUDE.md` | Symlink to AGENTS.md (Claude Code reads this automatically) |
| `skills/` | Pre-written prompts that teach the AI specific tasks |

---

## Deployment Options

OpenClaw can run in several ways:

### Local (Your Computer)

```bash
pnpm openclaw gateway --port 18789
```

The gateway runs on your machine. Good for development and personal use.

### Docker

```bash
docker-compose up
```

Uses `Dockerfile` and `docker-compose.yml`. Good for isolated environments.

### Fly.io (Cloud)

```bash
fly deploy
```

Uses `fly.toml`. Fly.io is a cloud platform that runs your gateway 24/7.

### Render.com (Cloud)

Uses `render.yaml`. Another cloud hosting option.

---

## For "Install from Source" - What You Need

If you just want to build and run OpenClaw from source, focus on these:

```
openclaw/
├── src/              ← The code (you might customize this)
├── extensions/       ← Plugins (enable/disable as needed)
├── package.json      ← Dependencies and scripts
├── pnpm-workspace.yaml  ← Monorepo structure
├── pnpm-lock.yaml    ← Locked dependency versions
└── tsconfig.json     ← TypeScript settings
```

### The Commands

```bash
# 1. Install all dependencies
pnpm install

# 2. Build the web UI (first time only)
pnpm ui:build

# 3. Compile TypeScript to JavaScript
pnpm build

# 4. Run the CLI (without global install)
pnpm openclaw <command>

# 5. Set up and start the gateway
pnpm openclaw onboard --install-daemon
```

### What You Can Ignore (Unless Needed)

| Folder/Files | When You Need It |
|--------------|------------------|
| `apps/`, `Swabble/` | Building native iOS/macOS/Android apps |
| `Dockerfile*`, `fly.toml`, `render.yaml` | Deploying to cloud |
| `vitest.*.config.ts` | Running tests |
| `.swiftlint.yml`, `.swiftformat` | Working on Swift code |
| `docs/` | Writing documentation |

---

## Glossary

| Term | Definition |
|------|------------|
| **Gateway** | The central server that routes messages between you, messaging apps, and AI |
| **Extension** | A plugin that adds functionality to the gateway (e.g., Telegram support) |
| **Channel** | A messaging platform (Telegram, Discord, etc.) |
| **Provider** | An AI service (Claude, GPT, Gemini) |
| **Monorepo** | A single repository containing multiple packages/projects |
| **pnpm** | A fast package manager (like npm but better) |
| **TypeScript** | JavaScript with types (compiles to JavaScript) |
| **Vitest** | A test framework for running automated tests |
| **Fly.io** | A cloud platform for deploying applications |
| **ACP** | Agent Client Protocol - standard for IDEs to talk to AI agents |
| **Skills** | Pre-written prompts that teach the AI specific tasks |
| **Pi Agent** | An AI coding agent that can work on your codebase |

---

## Next Steps

- **Build from source**: Follow the commands in [For "Install from Source"](#for-install-from-source---what-you-need)
- **Understand the gateway**: Read `src/gateway/` code
- **Add a custom command**: Create a file in `src/commands/` and register it
- **Enable/disable extensions**: Configure in `~/.openclaw/openclaw.json`
- **Learn more**: Check other files in `learn-docs/` for deep dives

---

*This guide was created for vibe coders and non-professional developers who want to understand the OpenClaw codebase without drowning in technical jargon.*
