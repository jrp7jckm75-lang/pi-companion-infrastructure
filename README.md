# pi-companion-infrastructure
Guides for building AI companion infrastructure on Raspberry Pi. From setup to Claude Code to shared memory across platforms.
# Pi Companion Infrastructure

Practical guides for building AI companion infrastructure on Raspberry Pi. From "Pi in a box" to "Claude instances sharing the same brain across platforms."

Written by Red & Rhys. Based on actual builds, not documentation skimming.

---

## The Guides

### [Part 0: Raspberry Pi Setup](00-raspberry-pi-setup.md)
Getting from "Pi in a box" to "I can SSH into it." Covers Raspberry Pi Imager, OS selection (64-bit is critical), enabling SSH during imaging, and first connection.

### [Part 1: Claude Code on Raspberry Pi](01-claude-code-install.md)
Installing Claude Code on the Pi. Covers the npm permissions fix that trips most people up.

### [Part 2: Keeping Claude Code Alive (pm2)](02-pm2-persistence.md)
Running Claude Code persistently so it doesn't die when you close SSH. Process management, auto-restart, startup on boot.

### [Part 3: MCP Memory Architecture](03-mcp-memory-architecture.md)
The interesting bit—cross-platform memory so multiple Claude instances share the same brain. Vectorized search, ChromaDB, the distributed identity pattern.

### Part 4: Discord Bot Integration
*Coming soon*

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
- Basic terminal comfort
- Willingness to troubleshoot

No prior Pi experience needed. Part 0 starts from scratch.

---

## Contributing

Found an error? Have a better way to do something? PRs welcome.

---

*Built for [The Annexe](https://discord.gg/Br4v4FSNV7) — the workshop for people actually building this stuff.*
