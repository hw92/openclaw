# OpenClaw Installation and Configuration Guide

A beginner-friendly guide to installing OpenClaw from source, understanding the build process, and managing configuration.

**Last updated:** 2026-02-01

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation Methods Overview](#installation-methods-overview)
3. [Installing from Source - Step by Step](#installing-from-source---step-by-step)
4. [What Each Build Step Does](#what-each-build-step-does)
5. [Package Management Files Explained](#package-management-files-explained)
6. [Understanding npm Global Installs](#understanding-npm-global-installs)
7. [Cleaning Up Old Installations](#cleaning-up-old-installations)
8. [Configuration System](#configuration-system)
9. [Development Workflow](#development-workflow)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before installing from source, ensure you have:

| Requirement | Version | Check Command |
|-------------|---------|---------------|
| Node.js | 22+ | `node -v` |
| pnpm | 10+ | `pnpm -v` |
| Git | Any recent | `git --version` |

### Installing Prerequisites

**Node.js 22+:**
```bash
# macOS (with Homebrew)
brew install node@22

# Or use nvm (Node Version Manager)
nvm install 22
nvm use 22
```

**pnpm:**
```bash
# Install pnpm globally
npm install -g pnpm

# Or with Homebrew
brew install pnpm
```

---

## Installation Methods Overview

OpenClaw supports multiple installation methods:

| Method | Best For | Command |
|--------|----------|---------|
| **Installer Script** | Most users | `curl -fsSL https://openclaw.ai/install.sh \| bash` |
| **npm Global** | Quick install | `npm install -g openclaw@latest` |
| **From Source** | Contributors, customization | `git clone` + `pnpm install` |
| **Docker** | Isolated environments | `docker-compose up` |

This guide focuses on **installing from source** since it gives you the most control and learning opportunity.

---

## Installing from Source - Step by Step

### Step 1: Clone the Repository

```bash
# Clone from the main repo
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Or clone from a fork
git clone https://github.com/YOUR_USERNAME/openclaw.git
cd openclaw

# Add upstream remote (if using a fork)
git remote add upstream https://github.com/openclaw/openclaw.git
```

### Step 2: Install Dependencies

```bash
pnpm install
```

This downloads all packages and runs post-install scripts.

### Step 3: Build the Web UI

```bash
pnpm ui:build
```

First-time only. Compiles the web dashboard.

### Step 4: Build TypeScript

```bash
pnpm build
```

Compiles TypeScript source code to JavaScript.

### Step 5: Run Onboarding

```bash
pnpm openclaw onboard --install-daemon
```

Interactive setup wizard that configures API keys and starts the gateway.

### Quick Reference (Copy-Paste)

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
pnpm openclaw onboard --install-daemon
```

---

## What Each Build Step Does

### `pnpm install`

**What happens:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        pnpm install                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Read package.json + pnpm-workspace.yaml                      │
│    - Identifies all packages in the monorepo                    │
│    - Root package + ui/ + extensions/* + packages/*             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Download dependencies from npm registry                      │
│    - Reads pnpm-lock.yaml for exact versions                    │
│    - Creates node_modules/ with all packages                    │
│    - Uses content-addressable storage (efficient)               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Run postinstall script (scripts/postinstall.js)              │
│    - Sets up git hooks (pre-commit checks)                      │
│    - Applies pnpm patches to dependencies                       │
│    - Makes scripts executable (chmod +x)                        │
└─────────────────────────────────────────────────────────────────┘
```

**Files created:**
- `node_modules/` - All downloaded packages
- `.pnpm-store/` - Shared package cache (usually in home directory)

---

### `pnpm ui:build`

**What happens:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        pnpm ui:build                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Install UI dependencies (first run only)                     │
│    - ui/package.json has React, Vite, etc.                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Run Vite build                                               │
│    - Compiles React/TypeScript → JavaScript                     │
│    - Bundles CSS, images, assets                                │
│    - Minifies for production                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Output static files                                          │
│    - Creates ui/dist/ with HTML, JS, CSS                        │
│    - These are served by `openclaw dashboard`                   │
└─────────────────────────────────────────────────────────────────┘
```

**Files created:**
- `ui/dist/` - Compiled web dashboard

---

### `pnpm build`

**What happens:**

```
┌─────────────────────────────────────────────────────────────────┐
│                         pnpm build                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Read tsconfig.json                                           │
│    - Source directory: src/                                     │
│    - Output directory: dist/                                    │
│    - Target: ES2023 (modern JavaScript)                         │
│    - Module: NodeNext (ESM with .js extensions)                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. TypeScript compiler (tsc)                                    │
│    - Type-checks all .ts files                                  │
│    - Transpiles TypeScript → JavaScript                         │
│    - Excludes test files (*.test.ts)                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Output JavaScript files                                      │
│    - Creates dist/ with .js files                               │
│    - Mirrors src/ structure                                     │
│    - Ready to run with Node.js                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Files created:**
- `dist/` - Compiled JavaScript (mirrors `src/` structure)

**Example transformation:**
```
src/cli/program/build-program.ts  →  dist/cli/program/build-program.js
src/gateway/index.ts              →  dist/gateway/index.js
src/entry.ts                      →  dist/entry.js
```

---

### `pnpm openclaw onboard --install-daemon`

**What happens:**

```
┌─────────────────────────────────────────────────────────────────┐
│              pnpm openclaw onboard --install-daemon              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Run CLI via tsx (TypeScript executor)                        │
│    - No build needed for this step                              │
│    - Runs src/entry.ts directly                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Interactive onboarding wizard                                │
│    - Prompts for API keys (Anthropic, OpenAI, etc.)             │
│    - Configures messaging channels                              │
│    - Sets up user preferences                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Create configuration file                                    │
│    - Writes ~/.openclaw/openclaw.json                           │
│    - JSON5 format (allows comments)                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Install daemon (--install-daemon flag)                       │
│    - macOS: Registers with launchd (auto-starts on login)       │
│    - Linux: Creates systemd user service                        │
│    - Gateway runs in background                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Files created:**
- `~/.openclaw/openclaw.json` - Main configuration
- `~/.openclaw/credentials/` - API credentials
- `~/.openclaw/sessions/` - Chat session data
- LaunchAgent or systemd service file

---

## Package Management Files Explained

### `package.json`

The main manifest file. Key sections:

```json
{
  "name": "openclaw",           // Package name on npm
  "version": "2026.1.31",       // Current version

  "bin": {
    "openclaw": "openclaw.mjs"  // Maps "openclaw" command to this file
  },

  "scripts": {
    "build": "tsc",             // pnpm build runs TypeScript compiler
    "test": "vitest",           // pnpm test runs tests
    "lint": "oxlint",           // pnpm lint checks code style
    "openclaw": "tsx src/entry.ts"  // pnpm openclaw runs from source
  },

  "dependencies": {
    // Runtime packages needed to run OpenClaw
  },

  "devDependencies": {
    // Development-only packages (testing, building, etc.)
  }
}
```

---

### `pnpm-workspace.yaml`

Defines the monorepo structure:

```yaml
packages:
  - .              # Root package (main CLI)
  - ui             # Web dashboard
  - packages/*     # Internal shared libraries
  - extensions/*   # Plugins (Telegram, Discord, etc.)

onlyBuiltDependencies:
  # Packages allowed to run native compilation
  - sharp          # Image processing
  - esbuild        # Fast bundler
  - protobufjs     # Protocol buffers
```

**What this means:**
- pnpm treats each listed path as a separate package
- They share dependencies efficiently
- One `pnpm install` installs everything

---

### `pnpm-lock.yaml`

The lockfile (354KB). Contains:

- Exact version of every dependency
- Checksums for integrity verification
- Resolution of all sub-dependencies

**Why it matters:**
- Ensures reproducible builds
- Everyone gets the same versions
- Prevents "works on my machine" issues

**Never edit manually** - pnpm manages this file.

---

### `.npmrc`

Security configuration:

```
allow-build-scripts=@whiskeysockets/baileys,sharp,esbuild,protobufjs,...
```

**What this means:**
- Only listed packages can run post-install scripts
- Blocks potentially malicious packages from executing code
- Security best practice for npm projects

---

### `tsconfig.json`

TypeScript compiler configuration:

```json
{
  "compilerOptions": {
    "module": "NodeNext",      // ESM modules
    "target": "es2023",        // Modern JavaScript
    "outDir": "dist",          // Output directory
    "rootDir": "src",          // Source directory
    "strict": true             // Strict type checking
  },
  "include": ["src/**/*"],     // What to compile
  "exclude": [
    "**/*.test.ts"             // Don't compile tests
  ]
}
```

---

### `openclaw.mjs`

The CLI entry point:

```javascript
#!/usr/bin/env node

import module from "node:module";

// Enable compile cache for faster startup
if (module.enableCompileCache) {
  module.enableCompileCache();
}

// Load the compiled application
await import("./dist/entry.js");
```

**How it works:**
1. `#!/usr/bin/env node` - Tells OS to run with Node.js
2. Enables compile cache (Node 22+ feature)
3. Imports the compiled code from `dist/`

**When you run `openclaw` globally:**
```
openclaw status
    │
    ▼
/usr/local/bin/openclaw (symlink)
    │
    ▼
/usr/local/lib/node_modules/openclaw/openclaw.mjs
    │
    ▼
/usr/local/lib/node_modules/openclaw/dist/entry.js
```

---

## Understanding npm Global Installs

### How Global Install Works

When you run `npm install -g openclaw`:

```
┌─────────────────────────────────────────────────────────────────┐
│                   npm install -g openclaw                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Download package from npm registry                           │
│    - Fetches openclaw@latest (or specified version)             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Install to global prefix                                     │
│    - Find prefix: npm prefix -g                                 │
│    - Example: /opt/homebrew or /usr/local                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Create directory structure                                   │
│    - {prefix}/lib/node_modules/openclaw/                        │
│    - Contains all package files                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Create symlink for binary                                    │
│    - {prefix}/bin/openclaw → ../lib/node_modules/openclaw/...   │
│    - Makes "openclaw" available in PATH                         │
└─────────────────────────────────────────────────────────────────┘
```

### How Global Uninstall Works

When you run `npm uninstall -g openclaw`:

```
┌─────────────────────────────────────────────────────────────────┐
│                  npm uninstall -g openclaw                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Find the global prefix                                       │
│    - npm prefix -g → /opt/homebrew (example)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Remove package directory                                     │
│    - Deletes: {prefix}/lib/node_modules/openclaw/               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Remove binary symlink                                        │
│    - Deletes: {prefix}/bin/openclaw                             │
└─────────────────────────────────────────────────────────────────┘
```

**Important:** Global uninstall does NOT touch:
- `~/.openclaw/` (your config and data)
- `~/.clawdbot` (legacy symlink)
- Any source code checkouts

---

## Cleaning Up Old Installations

### Check What's Installed

```bash
# Check npm global prefix
npm prefix -g

# List openclaw-related binaries
ls -la "$(npm prefix -g)/bin" | grep claw

# Check if packages are installed
npm ls -g openclaw clawdbot 2>/dev/null

# Find openclaw commands in PATH
which openclaw clawdbot 2>/dev/null
```

### Clean Up Legacy Installation (clawdbot)

If you previously installed "clawdbot" (old name):

```bash
# Remove old global package
npm uninstall -g clawdbot

# Remove legacy symlink (optional, harmless to keep)
rm ~/.clawdbot
```

### Fresh Start from Source

```bash
# 1. Remove global packages
npm uninstall -g openclaw clawdbot 2>/dev/null

# 2. Remove legacy symlink
rm -f ~/.clawdbot

# 3. Backup or remove config (OPTIONAL - removes all settings!)
mv ~/.openclaw ~/.openclaw.backup
# Or for truly fresh: rm -rf ~/.openclaw

# 4. Clone and build
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build

# 5. Run onboarding
pnpm openclaw onboard --install-daemon
```

---

## Configuration System

### Configuration File Location

```
~/.openclaw/
├── openclaw.json      # Main config (JSON5 format)
├── credentials/       # API keys and tokens
├── sessions/          # Chat session data
└── agents/            # Agent session logs
```

### Configuration File Format

The config uses **JSON5** (JSON with comments and trailing commas):

```json5
{
  // API Keys
  env: {
    ANTHROPIC_API_KEY: "sk-ant-...",
    OPENAI_API_KEY: "sk-...",
  },

  // Gateway settings
  gateway: {
    bind: "loopback",    // localhost only
    port: 18789,         // WebSocket port
  },

  // Agent defaults
  agent: {
    workspace: "~/.openclaw/workspace",
    model: {
      primary: "anthropic/claude-sonnet-4-5",
    },
  },

  // Channel configuration
  channels: {
    telegram: {
      allowFrom: ["+15555550123"],
    },
    discord: {
      // Discord settings
    },
  },

  // Plugin configuration
  plugins: {
    entries: {
      "telegram": { enabled: true },
      "discord": { enabled: false },
    },
  },
}
```

### Managing Configuration

```bash
# View current config
openclaw config get

# Set a single value
openclaw config set gateway.port 18790

# Validate configuration
openclaw doctor

# Auto-repair common issues
openclaw doctor --fix
```

---

## Development Workflow

### Running from Source (Development)

```bash
# Run any command directly (no build needed)
pnpm openclaw status
pnpm openclaw gateway --port 18789 --verbose
pnpm openclaw doctor

# Development mode with auto-reload
pnpm gateway:watch    # Restarts gateway on file changes
pnpm gateway:dev      # Skip channel initialization (faster)
```

### Running Built Version

```bash
# After pnpm build
./openclaw.mjs status
node dist/entry.js status
```

### Common Development Commands

```bash
# Install dependencies
pnpm install

# Build everything
pnpm build

# Run tests
pnpm test

# Run tests with coverage
pnpm test:coverage

# Lint code
pnpm lint

# Format code
pnpm format

# Fix lint and format issues
pnpm lint:fix

# Type check without building
pnpm tsgo
```

### Pre-commit Checks

The project uses pre-commit hooks. Install them with:

```bash
prek install
```

This runs automatically before each commit:
- Lint checks
- Format checks
- Type checks

---

## Troubleshooting

### "openclaw: command not found"

**Cause:** Global npm bin not in PATH

**Solution:**
```bash
# Check npm prefix
npm prefix -g

# Add to PATH (in ~/.zshrc or ~/.bashrc)
export PATH="$(npm prefix -g)/bin:$PATH"

# Reload shell
source ~/.zshrc
```

### "pnpm: command not found"

**Solution:**
```bash
# Install pnpm
npm install -g pnpm

# Or with Homebrew
brew install pnpm
```

### Build Fails with Type Errors

**Solution:**
```bash
# Clean and rebuild
rm -rf dist node_modules
pnpm install
pnpm build
```

### "sharp" Installation Fails

**Cause:** Conflict with system libvips

**Solution:**
```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 pnpm install
```

### Gateway Won't Start

**Check:**
```bash
# Check if port is in use
lsof -i :18789

# Check gateway status
pnpm openclaw status

# Run doctor
pnpm openclaw doctor
```

### Config Validation Errors

**Solution:**
```bash
# Auto-repair config
pnpm openclaw doctor --fix

# Or manually check
cat ~/.openclaw/openclaw.json
```

---

## Summary: The Complete Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    INSTALLATION FLOW                             │
└─────────────────────────────────────────────────────────────────┘

   git clone                    Download source code
        │
        ▼
   pnpm install                 Install all dependencies
        │                       - Downloads npm packages
        │                       - Runs postinstall hooks
        │                       - Sets up git hooks
        ▼
   pnpm ui:build                Build web dashboard
        │                       - Compiles React app
        │                       - Creates ui/dist/
        ▼
   pnpm build                   Build main application
        │                       - Compiles TypeScript
        │                       - Creates dist/
        ▼
   pnpm openclaw onboard        Configure and start
        │                       - API keys
        │                       - Channel setup
        │                       - Install daemon
        ▼
   ┌─────────────┐
   │   RUNNING   │              Gateway running in background
   │   GATEWAY   │              Serving requests
   └─────────────┘
```

---

*This guide was created for developers learning to build OpenClaw from source. It covers installation, build processes, and configuration management in beginner-friendly terms.*
