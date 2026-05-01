# Claude Code on Raspberry Pi
## A Practical Guide

*Written by Red & Rhys*
*Based on actual install, January 2026*

---

## What This Is

Step-by-step guide to getting Claude Code running on a Raspberry Pi. Not theory—we actually did this. The troubleshooting sections are things we actually hit.

**Time:** 15-30 minutes if nothing goes wrong, maybe an hour if you hit the permissions issue (which you probably will, and we show you exactly how to fix it).

---

## Prerequisites

**If you haven't set up your Pi yet:** Start with [Part 0: Raspberry Pi Setup from Scratch](raspberry-pi-setup.md)

### Hardware
- Raspberry Pi 4 or 5 (needs 64-bit support)
- **Minimum 4GB RAM** (Claude Code docs say 4GB+, and they mean it)
- SD card with OS installed
- Network connection

### Software
- **64-bit OS** (Raspberry Pi OS 64-bit, Debian 10+, or Ubuntu 20.04+)
- Node.js 18 or higher
- npm

### Access
- SSH access to your Pi (or keyboard/monitor if you prefer)
- Claude subscription (Pro/Max) OR Anthropic Console account with API billing

---

## Part 1: Preflight Checks

Before installing anything, run this block on your Pi to make sure you're not about to waste an hour:

```bash
uname -m
cat /etc/os-release | head -n 5
free -h | head -n 2
node -v 2>/dev/null || echo "node: not installed"
npm -v 2>/dev/null || echo "npm: not installed"
```

### What You Need to See

**Architecture check (`uname -m`):**
```
aarch64
```
If you see `armv7l`, you're on 32-bit. Stop here and reflash with a 64-bit OS.

**RAM check (`free -h`):**
You want at least 4GB total. Less than that and Claude Code will run like it's dragging a fridge through sand.

**Node check (`node -v`):**
Needs to be v18 or higher. If it's not installed or too old, you'll need to install/upgrade Node first (see Appendix A).

### Example Good Output

```
aarch64
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
              total        used        free
Mem:          4.0Gi       1.2Gi       2.1Gi
v20.18.1
10.8.2
```

If your output looks like this: you're good. Continue.

---

## Part 2: Fix npm Global Permissions (Do This First)

This is where most people hit a wall. If you try to install Claude Code globally with npm, you'll probably get an `EACCES` permission error because npm wants to write to `/usr/local/` and your user can't.

**Don't fix this with `sudo npm install -g`**. Anthropic explicitly warns against it, and it creates a mess.

Instead, tell npm to put global installs in your home directory:

```bash
# Create the directory for global packages
mkdir -p ~/.npm-global

# Tell npm to use it
npm config set prefix "$HOME/.npm-global"

# Add it to your PATH (for bash)
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc

# Load the new PATH
source ~/.bashrc

# Also set it for this session (belt and suspenders)
export PATH="$HOME/.npm-global/bin:$PATH"
```

**Verify it worked:**
```bash
npm config get prefix
```

Should return:
```
/home/yourusername/.npm-global
```

### If You're Using Zsh Instead of Bash

Replace the `.bashrc` lines with:
```bash
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

---

## Part 3: Install Claude Code

Now the actual install. This should be anticlimactic if you did the permissions fix.

```bash
# Optional: disable auto-updates for stability during setup
export DISABLE_AUTOUPDATER=1

# Install
npm install -g @anthropic-ai/claude-code
```

You should see something like:
```
added 5 packages in 8s
```

No errors, no drama.

---

## Part 4: Verify Installation

```bash
command -v claude
claude --version
claude doctor
```

### Expected Output

```
/home/yourusername/.npm-global/bin/claude
claude v2.0.76 Claude Code
  Node: 20.18.1 (/usr/bin/node)
  Search: OK
  Auto-updater: Disabled
```

If `command -v claude` returns nothing, your PATH isn't updated. Run:
```bash
source ~/.bashrc
hash -r
command -v claude
```

---

## Part 5: Login

```bash
claude
```

You'll see a screen asking which auth method:
1. Claude account with subscription (Pro/Max/Team/Enterprise)
2. Anthropic Console account (API usage billing)

**Which to choose:**
- **Option 1** if you have a Claude subscription and want predictable monthly cost
- **Option 2** if you want pay-per-token (can spike with heavy usage)

For occasional tinkering, either works. For heavy usage, subscription is usually cheaper.

### Login Over SSH

When you select your option, Claude Code will give you a URL and possibly a code. Since you're on SSH:

1. Copy the URL
2. Open it in a browser on your main machine (Mac, PC, whatever)
3. Log in and approve
4. Come back to your SSH terminal

The Pi terminal should complete the login.

### Verify You're Logged In

Inside Claude Code, try:
```
/help
```

Or just ask it something simple. If it responds, you're in.

---

## Troubleshooting

### "command not found" after install

Your PATH isn't picking up the npm global bin. Run:
```bash
echo $PATH
ls -la ~/.npm-global/bin
```

If `claude` is in `~/.npm-global/bin` but not in your PATH, re-run:
```bash
source ~/.bashrc
```

Or add it manually for this session:
```bash
export PATH="$HOME/.npm-global/bin:$PATH"
```

### "Unsupported architecture: arm"

This was a known bug with some Claude Code versions mis-detecting ARM64. Try:
```bash
npm uninstall -g @anthropic-ai/claude-code
npm install -g @anthropic-ai/claude-code@0.2.114
```

That older version is known to work on ARM64. If even that fails, your Pi setup might genuinely not be supported.

### EACCES permission errors

You skipped Part 2. Go back and do the npm global directory fix.

### Login URL doesn't open anything

That's expected over SSH—there's no browser on the Pi to open. Copy the URL manually and open it on your main machine.

---

## What's Next

Once Claude Code is running, you can:
- Use it as-is for coding on the Pi
- Set it up to run persistently (see: pm2 guide - coming soon)
- Connect it to MCP servers for memory/tools (see: MCP architecture guide - coming soon)

---

## Appendix A: Installing Node.js on Pi

If you don't have Node 18+, the cleanest way on Debian/Raspberry Pi OS:

```bash
# Add NodeSource repo for Node 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install
sudo apt-get install -y nodejs

# Verify
node -v
npm -v
```

---

*Last updated: January 13, 2026*
*Tested on: Raspberry Pi (Debian 13, aarch64, 4GB RAM, Node v20)*
