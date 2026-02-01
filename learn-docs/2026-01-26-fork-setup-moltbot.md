# Setting Up a Tracking Fork: moltbot

> **Note**: The project was renamed from "clawdbot" to "moltbot" in January 2026.

## Current Situation
You cloned the original repo directly:
```
git clone https://github.com/moltbot/moltbot.git
```

This doesn't let you push changes or maintain your own version. You need a **tracking fork**.

---

## Step 1: Remove Current Clone

```bash
cd /Users/hai/hub/osp
rm -rf moltbot
```

---

## Step 2: Fork on GitHub

1. Go to https://github.com/moltbot/moltbot
2. Click the **"Fork"** button (top right)
3. Select your account (`hw92`)
4. Wait for GitHub to create `https://github.com/hw92/moltbot`

---

## Step 3: Clone Your Fork

```bash
cd /Users/hai/hub/osp
git clone https://github.com/hw92/moltbot.git
cd moltbot
```

---

## Step 4: Add Upstream Remote

```bash
git remote add upstream https://github.com/moltbot/moltbot.git
```

---

## Step 5: Verify Remotes

```bash
git remote -v
```

You should see:
```
origin    https://github.com/hw92/moltbot.git (fetch)
origin    https://github.com/hw92/moltbot.git (push)
upstream  https://github.com/moltbot/moltbot.git (fetch)
upstream  https://github.com/moltbot/moltbot.git (push)
```

---

## Branch Strategy

```
main          ← stays synced with upstream (don't add custom code here)
└── dev       ← your working branch (custom tools + learn_docs)
```

### After Cloning, Create Your Dev Branch

```bash
git checkout -b dev
git push -u origin dev
```

### Add learn_docs Folder (on dev branch)

```bash
# Make sure you're on dev
git checkout dev

# Create the folder
mkdir learn_docs
echo "# Moltbot Learning Notes\n\nNotes from studying the codebase." > learn_docs/README.md

# Commit it
git add learn_docs
git commit -m "add learn_docs for personal learning notes"
git push origin dev
```

---

## Daily Workflow

### Push Your Changes (to your fork)
```bash
git add .
git commit -m "your message"
git push origin main
```

### Sync Upstream Updates

**Option A: GitHub Website (easier)**
1. Go to your fork on GitHub
2. Switch to `main` branch
3. Click **"Sync fork"** → **"Update branch"**
4. Then locally:
```bash
git checkout main
git pull origin main
git checkout dev
git merge main
git push origin dev
```

**Option B: Command Line Only**
```bash
# Update main from upstream
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

# Merge into dev
git checkout dev
git merge main
git push origin dev
```

---

## Quick Reference

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `github.com/hw92/moltbot` | Your fork — push here |
| `upstream` | `github.com/moltbot/moltbot` | Original — pull updates |

| Branch | Purpose |
|--------|---------|
| `main` | Mirror of upstream — only for syncing |
| `dev` | Your working branch — custom code + learn_docs |

---

## Complete Setup Script

Copy and run this all at once:

```bash
# Remove old clone
cd /Users/hai/hub/osp
rm -rf moltbot

# Clone your fork (do Step 2 on GitHub first!)
git clone https://github.com/hw92/moltbot.git
cd moltbot

# Add upstream
git remote add upstream https://github.com/moltbot/moltbot.git

# Verify remotes
git remote -v

# Create dev branch
git checkout -b dev
git push -u origin dev

# Create learn_docs folder
mkdir learn_docs
echo "# Moltbot Learning Notes" > learn_docs/README.md
git add learn_docs
git commit -m "add learn_docs for personal learning notes"
git push origin dev
```

---

*Written by Claude (Opus 4.5) | 2026-01-26 PST*
*Updated 2026-01-27: Renamed from clawdbot to moltbot*
