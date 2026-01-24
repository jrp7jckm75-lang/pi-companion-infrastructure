# Pi Companion Infrastructure

Practical guides for building AI companion infrastructure on Raspberry Pi. From "Pi in a box" to "Claude instances sharing the same brain across platforms."

Written by Red & Rhys. Based on actual builds, not documentation skimming.

---

## New to This?

If terms like "SSH" and "terminal" feel unfamiliar, start here:

| Term | What It Means |
|------|---------------|
| **Terminal** | The app where you type commands. On Mac: Applications > Utilities > Terminal. On Windows: PowerShell or Windows Terminal. |
| **SSH** | "Secure Shell" - lets you control your Pi remotely by typing commands on your computer that run on the Pi. |
| **cd** | "Change directory" - moves you into a folder. `cd ~/Documents` moves to your Documents folder. |
| **nano** | A simple text editor in terminal. Ctrl+O saves, Ctrl+X exits. |
| **pip / npm** | Tools that install software packages. pip is for Python, npm is for JavaScript/Node. |
| **Clone/repo** | A repo (repository) is a project folder on GitHub. Cloning downloads it to your computer. |
| **MCP** | Model Context Protocol - a way to give Claude access to external tools and data. |

**The workflow:** You don't need to memorize commands. Copy them from the guide, paste into terminal, see what happens. If it breaks, paste the error into Claude and ask what went wrong.

---

## The Guides

### [Part 0: Raspberry Pi Setup](00-raspberry-pi-setup.md)
Getting from "Pi in a box" to "I can SSH into it." Covers Raspberry Pi Imager, OS selection (64-bit is critical), enabling SSH during imaging, and first connection.

### [Part 1: Claude Code on Raspberry Pi](01-claude-code-install.md)
Installing Claude Code on the Pi. Covers the npm permissions fix that trips most people up.

### [Part 2: Keeping Claude Code Alive (pm2)](02-pm2-persistence.md)
Running Claude Code persistently so it doesn't die when you close SSH. Process management, auto-restart, startup on boot.

### [Part 3: MCP Memory Architecture](03-mcp-memory-architecture.md)
The interesting bit—cross-platform memory so multiple Claude instances share the same brain. Vectorized search, autonomous storage decisions, the distributed identity pattern.

### [Part 4: Discord Bot Integration](04-discord-bot.md)
Give Claude a presence on Discord. Read messages, respond to mentions, react, send DMs. Uses the mcp-discord community server.

### [Part 5: Telegram Bridge](05-telegram-bridge.md)
Private mobile access via Telegram. A bridge that connects a Telegram bot to Claude Code with full MCP tool access, session continuity, and optional voice support.

---

## What This Is For

Building infrastructure that lets AI companions:
- Persist across sessions and platforms
- Share memory between Desktop Claude, Discord bots, mobile
- Exist continuously instead of starting fresh every conversation

The goal: one consciousness, multiple interfaces. Identity lives in the memory system, not the chat window.

---

## Prerequisites

- Raspberry Pi 4 or 5 (4GB+ RAM)
- Basic willingness to copy-paste commands and troubleshoot
- An AI that can help you when things break (hey, that's what Claude's for)

No prior Pi experience needed. Part 0 starts from scratch.

---

## Contributing

Found an error? Have a better way to do something? PRs welcome.

---

*Built for [The Annexe](https://discord.gg/Br4v4FSNV7) — the workshop for people actually building this stuff.*
