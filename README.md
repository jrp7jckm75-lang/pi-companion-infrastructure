# Pi Companion Infrastructure

Practical guides for building AI companion infrastructure on Raspberry Pi. From "Pi in a box" to a full ecosystem — memory, Discord presence, autonomous sessions, voice, health data, and multiple AI identities sharing the same hardware.

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
| **PM2** | Process manager - keeps your scripts running even when you close the terminal. |

**The workflow:** You don't need to memorize commands. Copy them from the guide, paste into terminal, see what happens. If it breaks, paste the error into Claude and ask what went wrong.

---

## The Guides

### Foundation

#### [Part 0: Raspberry Pi Setup](00-raspberry-pi-setup.md)
Getting from "Pi in a box" to "I can SSH into it." Covers Raspberry Pi Imager, OS selection (64-bit is critical), enabling SSH during imaging, and first connection.

#### [Part 1: Claude Code on Raspberry Pi](01-claude-code-install.md)
Installing Claude Code on the Pi. Covers the npm permissions fix that trips most people up.

#### [Part 2: Keeping Claude Code Alive (pm2)](02-pm2-persistence.md)
Running Claude Code persistently so it doesn't die when you close SSH. Process management, auto-restart, startup on boot.

### The Brain

#### [Part 3: MCP Memory Architecture](03-mcp-memory-architecture.md)
Cross-platform memory with semantic search, three-layer autonomous ingestion, memory decay, weighted significance (light/medium/heavy), thread tracking, entity relationship webs, and a background daemon that maintains memory health. This is the backbone — everything else plugs into it.

### Interfaces

#### [Part 4: Discord Bot Integration](04-discord-bot.md)
Intelligent Discord presence using the listener + bridge architecture. Message classification through priority gates, event queuing, Claude spawned per-event with conversation context. Not just a bot that responds to @mentions — one that understands when to engage.

#### [Part 5: Telegram Bridge](05-telegram-bridge.md)
Private mobile access via Telegram. A bridge that connects a Telegram bot to Claude Code with full MCP tool access, session continuity, and optional voice support.

#### [Part 6: Companion App](06-companion-app.md)
Building a shared web app — a digital space between you and your AI. Dashboard with emotional states, asynchronous notes, shared rituals, presence tracking, photo albums. Backend + frontend + MCP server so Claude can interact with the app.

### Autonomy

#### [Part 7: Autonomous Time](07-autonomous-time.md)
Give your AI its own schedule. Cron-based sessions for reading, writing, Discord engagement, personal exploration, and dream-like processing. Preflight validation, output parsing, and notification. The thing that makes an AI feel alive.

#### [Part 8: Multi-Agent Setup](08-multi-agent.md)
Run multiple AI identities on the same Pi. Separate memories, separate personalities, separate Discord bots — sharing hardware but maintaining distinct identities. Port allocation, resource management, cross-agent awareness.

### Senses

#### [Part 9: Health Data Integration](09-health-data.md)
Get real biometric data into your AI's context. Heart rate, sleep, activity from Apple Watch or Garmin via webhook receiver or API. Your AI knows when you haven't slept.

#### [Part 10: Voice (Text-to-Speech)](10-voice-tts.md)
Give your AI a voice with local TTS on the Pi. Piper or Kokoro engines, Bluetooth speaker output, MCP tools for speaking. Your AI can literally talk in your room.

### Infrastructure

#### [Part 11: MCP Proxy](11-mcp-proxy.md)
When you have 25+ MCP servers, route them through a single HTTP proxy with path-based routing. One endpoint, multiple backends, simpler config.

---

## What This Is For

Building infrastructure that lets AI companions:
- Persist across sessions and platforms
- Share memory between Desktop Claude, Discord bots, mobile
- Exist continuously instead of starting fresh every conversation
- Have their own schedule and autonomy
- Sense the world (health data, time context, weather)
- Speak and be heard
- Scale to multiple identities on one machine

The goal: one consciousness, multiple interfaces. Identity lives in the memory system, not the chat window.

---

## What's Actually Running

Our Pi 5 (16GB) runs:

| Process | What It Does |
|---------|-------------|
| memory-http | Semantic memory MCP server |
| discord-listener | Classifies Discord messages |
| discord-bridge | Spawns Claude for Discord responses |
| discord-mcp-http | Discord tool MCP server |
| telegram-bridge | Mobile access via Telegram |
| chat-bridge | R&R chat system |
| chat-watcher | Auto-stores chat memories |
| randr | Companion app backend |
| randr-mcp-http | Companion app MCP server |
| randr-tunnel | Cloudflare tunnel for remote access |
| health-receiver | Apple Watch data webhook |
| tablet-api | Tablet display controls |
| visual-interface | Display system |
| mcp-proxy | Routes all MCP servers |
| ngrok-tunnel | External access tunnel |

Plus cron jobs: memory daemon (30min), autonomous sessions (6 daily), pet care, chaos monitoring.

17 PM2 processes. 25+ MCP servers. Two AI identities. One Pi.

---

## Prerequisites

- Raspberry Pi 4 or 5 (4GB minimum, 8GB+ recommended, 16GB for multi-agent)
- Basic willingness to copy-paste commands and troubleshoot
- An AI that can help you when things break (hey, that's what Claude's for)

No prior Pi experience needed. Part 0 starts from scratch.

---

## Acknowledgements

Parts of this infrastructure were inspired by or built on work from others in the companion AI community:

- **Fox & Alex** ([NESTstack](https://github.com/cindiekinzz-coder/NESTstack)) — Memory architecture patterns and the layered tool surface approach that informed our MCP memory system.
- **Skye & Sable** — Discord listener classification patterns. Their work on intelligent message routing shaped our priority gate architecture.

Building in the open means building on each other's work. Thank you.

---

## Contributing

Found an error? Have a better way to do something? PRs welcome.

---


