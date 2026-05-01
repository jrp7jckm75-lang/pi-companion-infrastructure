# Part 2: Process Management with PM2

Keep your services running after you close SSH. PM2 is the process manager that holds everything together — not just Claude Code, but every service you'll build in this series.

---

## The Problem

You install Claude Code on your Pi. You run it. You close your SSH session. It dies.

Same thing happens with your memory server, your Discord bot, your Telegram bridge, your companion app. Without a process manager, nothing survives you closing the terminal.

PM2 solves this. It:
- Keeps processes running after you disconnect
- Restarts them if they crash
- Starts them automatically on boot
- Gives you logs and monitoring for everything

---

## Prerequisites

- Claude Code already installed and working (see Part 1)
- Node.js (you have this if you installed Claude Code via npm)

---

## Install PM2

```bash
npm install -g pm2
```

If you followed Part 1's npm permissions fix, this should just work. If you get EACCES errors, go back and do the `~/.npm-global` setup.

**Verify:**
```bash
pm2 --version
```

---

## Starting Your First Process

Start Claude Code under PM2:

```bash
pm2 start ~/.npm-global/bin/claude --name "claude-code"
```

> **Always use the full path** for the executable. PM2 runs with a different PATH than your shell — `pm2 start claude` will often fail with "command not found."

Check it's running:

```bash
pm2 list
```

```
┌─────┬──────────────┬──────────┬────────┐
│ id  │ name         │ status   │ uptime │
├─────┼──────────────┼──────────┼────────┤
│ 0   │ claude-code  │ online   │ 5s     │
└─────┴──────────────┴──────────┴────────┘
```

---

## Essential Commands

```bash
# See what's running
pm2 list

# View logs (real-time, all processes)
pm2 logs

# View logs for one process
pm2 logs claude-code

# View last 100 lines
pm2 logs claude-code --lines 100

# Restart a process
pm2 restart claude-code

# Stop a process (stays in PM2 list)
pm2 stop claude-code

# Remove from PM2 entirely
pm2 delete claude-code

# Show detailed info about a process
pm2 show claude-code
```

---

## Starting on Boot

This is the important bit. You want your services to start automatically when the Pi boots or after a power cut.

```bash
# Generate the startup script
pm2 startup
```

This prints a command. **Copy and run the exact command it outputs.** It'll look something like:

```bash
sudo env PATH=$PATH:/home/username/.npm-global/bin pm2 startup systemd -u username --hp /home/username
```

Then save your current process list:

```bash
pm2 save
```

Now when your Pi reboots, PM2 will restore everything. Test it:

```bash
sudo reboot
```

Wait for it to come back, SSH in, run `pm2 list`. Everything should be online.

> **Important:** Run `pm2 save` after every change to your process list (starting, stopping, or deleting processes). Otherwise the saved state won't match what's running.

---

## Managing Multiple Services

As you build through this series, you'll accumulate services. Here's what a typical setup looks like:

```bash
# Memory server (Part 3)
pm2 start ~/memory-server/venv/bin/python3 \
  --name memory-http \
  -- ~/memory-server/server.py

# Discord listener (Part 4)
pm2 start ~/discord-bot/listener.js --name discord-listener

# Discord bridge (Part 4)
pm2 start ~/discord-bot/bridge.js --name discord-bridge

# Discord MCP server (Part 4)
pm2 start ~/mcp-servers/discord/start.sh --name discord-mcp-http

# Telegram bridge (Part 5)
pm2 start ~/mcp-servers/telegram-bridge/venv/bin/python3 \
  --name telegram-bridge \
  -- ~/mcp-servers/telegram-bridge/bridge.py

# Companion app (Part 6)
pm2 start ~/companion-app/backend/server.js --name companion-app

pm2 save
```

### Tips for Multiple Processes

**Name things clearly.** You'll be reading `pm2 list` a lot. Names like `discord-listener` and `discord-bridge` tell you what's what at a glance.

**Python processes need the full venv path.** Don't just `pm2 start python3` — use the venv's Python directly so it finds all your installed packages:

```bash
pm2 start ~/myproject/venv/bin/python3 --name my-service -- ~/myproject/script.py
```

**Shell scripts work too.** For services that need environment setup before starting:

```bash
pm2 start ~/myproject/start.sh --name my-service --interpreter bash
```

**Non-Node processes need `--interpreter none`** if PM2 tries to run them through Node:

```bash
pm2 start /usr/local/bin/some-binary --name my-service --interpreter none -- --flag value
```

---

## Logs

### Where They Live

PM2 stores logs in `~/.pm2/logs/`. Each process gets two files:
- `process-name-out.log` — standard output
- `process-name-error.log` — errors

### Watching Logs

```bash
# All processes at once
pm2 logs

# One process
pm2 logs discord-bridge

# Just errors
pm2 logs discord-bridge --err

# Last 50 lines (not streaming)
pm2 logs discord-bridge --lines 50 --nostream
```

### Log Rotation

Logs can get big — especially if you're running 10+ services. Install the rotation module:

```bash
pm2 install pm2-logrotate
```

Default settings are fine. It'll automatically rotate and compress old logs.

---

## Monitoring

### Quick Check

```bash
pm2 list
```

Shows all processes with status, uptime, restart count, CPU, and memory.

### Detailed View

```bash
pm2 show process-name
```

Shows PID, script path, args, environment, log paths, and more.

### Interactive Dashboard

```bash
pm2 monit
```

Real-time CPU, memory, and logs for all processes. Press `q` to exit.

---

## Troubleshooting

### Process keeps restarting (restart loop)

Check the error logs:
```bash
pm2 logs process-name --err --lines 50
```

Common causes:
- Missing environment variables
- Wrong file path
- Python package not installed in the venv
- Port already in use (another process is on the same port)

### Process shows "errored" or "stopped"

```bash
# Check what happened
pm2 show process-name

# Try starting it manually to see the error
pm2 stop process-name
# Then run the command directly to see output:
node ~/discord-bot/listener.js
```

### High memory usage

The Pi has limited RAM. Check what's using what:

```bash
pm2 list
```

The `mem` column shows each process's usage. If something is eating too much:

```bash
# Restart it (clears memory)
pm2 restart process-name
```

For Python services with memory leaks, you can set auto-restart on memory threshold:

```bash
pm2 start script.py --name my-service --max-memory-restart 300M
```

### "pm2 startup" not working after reboot

Re-run the setup:
```bash
pm2 unstartup
pm2 startup
# Run the command it outputs
pm2 save
```

### Processes running but not accessible

Check the process is actually listening:
```bash
pm2 logs process-name --lines 20
ss -tlnp | grep PORT_NUMBER
```

---

## What's Next

With PM2 running, your Pi can hold multiple services persistently. Everything you build from here — memory, Discord, Telegram, companion app, autonomous sessions — will run under PM2.

Part 3 covers the memory system — how to give your AI persistent memory across sessions and platforms.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
