# OpenClaw Installation Deep Dive: Global vs Source

**Date:** 2026-02-01

This document explains the differences between the two primary ways to install and run OpenClaw: the **Recommended Installer (Global)** and **Building from Source**. It also clarifies where files live and how to properly uninstall them.

## 1. The Two Installation Methods

### A. Recommended Installer (`curl`)
**Command:** `curl -fsSL https://openclaw.ai/install.sh | bash`

*   **Mechanism:** Installs `openclaw` as a **global npm package**.
*   **Binary Location:** Your system's global node bin path (e.g., `/usr/local/bin/openclaw`, `/opt/homebrew/bin/openclaw`).
*   **Source Code Location:** Buried deep in global `node_modules` (e.g., `/usr/local/lib/node_modules/openclaw/dist`).
*   **Execution:** When you run `openclaw`, your shell finds it in `$PATH`.
*   **Best For:** Users who just want to *run* the bot.

### B. Building from Source (Dev)
**Command:** `git clone ... && pnpm install && pnpm build`

*   **Mechanism:** Compiles TypeScript code into JavaScript within your local project folder.
*   **Binary Location:** Inside your project folder at `./dist`.
*   **Source Code Location:** Your local `src` folder.
*   **Execution:** 
    *   You run it via `pnpm openclaw` (which maps to `node scripts/run-node.mjs`).
    *   It executes the *local* compiled code from `./dist`.
*   **Best For:** Developers who want to learn, modify, or contribute to the codebase.

| Feature | Global Install (`curl`) | Source Build (Local) |
| :--- | :--- | :--- |
| **Command** | `openclaw` | `pnpm openclaw` |
| **Updates** | `npm update -g openclaw` | `git pull && pnpm build` |
| **Modifiable?** | No (Read-only distribution) | Yes (Edit, Rebuild, Run) |
| **Dependency** | System Global Node/NPM | Local `node_modules` |

---

## 2. Configuration & State

Regardless of how you install the *binary*, both methods use the **same default location** for configuration and state:

*   **Config File:** `~/.openclaw/openclaw.json`
*   **State/Logs:** `~/.openclaw/`

This means you can theoretically switch between a global install and a source build, and they will share the same settings (unless you explicitly point them elsewhere).

---

## 3. Uninstalling Properly

The cleanup process differs slightly because the binary location is different, but the *service* cleanup is the same.

### The "Service" (Daemon)
OpenClaw often runs as a background service (daemon). You must stop this **before** deleting files.

**The Golden Rule:** 
> **Stop the engine (service) before you scrap the car (files).**

### Uninstalling Global Install
1.  **Run Uninstaller:** `openclaw uninstall --all`
    *   Stops the daemon service.
    *   Deletes `~/.openclaw` (Config/State).
    *   Runs `npm uninstall -g openclaw` to remove the binary.

### Uninstalling Source Build
1.  **Run Local Uninstaller:** `pnpm openclaw uninstall`
    *   Stops the daemon service.
    *   Deletes `~/.openclaw` (Config/State).
    *   *Note: This DOES NOT delete your source code folder.*
2.  **Delete Project:** Manually delete your project folder.
    *   `rm -rf /path/to/openclaw`

---

## 4. Key Takeaways

1.  **Build Output:** `pnpm build` creates a `dist/` folder in your project root. This is where the actual executable code lives.
2.  **`pnpm` Magic:** Running `pnpm openclaw` is a shortcut that tells Node to execute your local project's entry script (`openclaw.mjs` -> `dist/index.js`).
3.  **Global vs Local:** Global relies on your shell's `PATH`. Local relies on `package.json` scripts.

---
**Written by:** Antigravity
**Timestamp:** 2026-02-01 20:40:29
