# OpenClaw Troubleshooting: Gateway Lifecycle & Authentication

**Date:** 2026-02-01

This lesson covers common pitfalls when managing the OpenClaw service and configuring authentication, based on real-world troubleshooting.

## 1. The Gateway Lifecycle (Stop vs. Install)

A common point of confusion is how the OpenClaw gateway service is managed on macOS (`launchd`). The commands `stop` and `start` behave differently than you might expect from other systems.

### The Behavior
*   **`openclaw gateway stop`**: This command **unloads** the service agent entirely from the system. It effectively "uninstalls" the running instance.
*   **`openclaw gateway start`**: This command attempts to load an *existing* service registration.

### The Trap
If you run `stop` and then try to run `start`, it will likely fail with:
> `Gateway service not loaded.`

This happens because `stop` removed the registration that `start` is looking for.

### The Solution: Correct Workflows
1.  **To Reboot (e.g., after config changes):**
    *   Use **`restart`**. This is safe and keeps the registration.
    *   Command: `pnpm openclaw gateway restart`

2.  **To Restore after a Stop:**
    *   Use **`install`**. This re-registers the service agent and starts it immediately.
    *   Command: `pnpm openclaw gateway install`

**Mental Model:**
*   `stop` = "Delete the runner."
*   `install` = "Create and start the runner."
*   `restart` = "Bounce the runner."

---

## 2. Authentication Errors ("Invalid Bearer Token")

If you see an error like this in your chat channel (WhatsApp/Telegram):
> `[openclaw] HTTP 401: authentication_error: Invalid bearer token`

It means your LLM provider (e.g., Anthropic/OpenAI) is rejecting your API key.

### Where is the Key?
OpenClaw separates **Configuration** (in `openclaw.json`) from **Secrets** (in `.env`).
*   `openclaw.json` says: "Use Anthropic profile."
*   `.env` says: "Here is the key: `sk-ant-...`"

### How to Fix
1.  **Edit the secrets file:**
    ```bash
    code ~/.openclaw/.env
    ```
2.  **Check the variable name:**
    *   Anthropic: `ANTHROPIC_API_KEY`
    *   OpenAI: `OPENAI_API_KEY`
3.  **Ensure validity:** Keys must be exact strings starting with the provider prefix (e.g., `sk-ant-`).
4.  **Restart the Gateway:** The change **will not take effect** until you restart the service.
    ```bash
    pnpm openclaw gateway restart
    ```

---

## 3. Configuring Channels (WhatsApp)

To add channels like WhatsApp, you don't edit config files manually.

1.  **Run the interactive setup:**
    ```bash
    pnpm openclaw configure --section channels
    ```
2.  **Scan the QR Code:**
    *   Open WhatsApp on your phone -> Linked Devices -> Link a Device.
3.  **Verify the Number:**
    *   The bot will link to whatever account scans the code.
    *   To verify which number is linked: Open WhatsApp on your phone -> Settings -> Profile (the number is listed under your name).
