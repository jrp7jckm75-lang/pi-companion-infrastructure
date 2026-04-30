# Keeping Claude Code Alive with pm2
## Part 2 of the Pi Infrastructure Series

*Written by Red & Rhys 
*January 2026*

---

## The Problem

You install Claude Code on your Pi. You run it. You close your SSH session. It dies.

That's not a persistent home—that's a waiting room.

pm2 solves this. It's a process manager that:
- Keeps Claude Code running after you disconnect
- Restarts it if it crashes
- Can start it automatically on boot
- Gives you logs and monitoring

---

## Prerequisites

- Claude Code already installed and working (see Part 1)
- Node.js (you have this if you installed Claude Code via npm)

---

## Install pm2

```bash
npm install -g pm2
```

If you followed Part 1's npm permissions fix, this should just work. If you get EACCES errors, go back and do the `~/.npm-global` setup.

**Verify:**
```bash
pm2 --version
```

---

## Running Claude Code Under pm2

### The Basic Command

```bash
pm2 start claude --name "claude-code"
```

This tells pm2 to:
- Start the `claude` command
- Name the process "claude-code" (so you can refer to it later)
- Keep it running

### Check It's Running

```bash
pm2 status
```

You should see something like:
```
┌─────┬──────────────┬─────────────┬─────────┬─────────┬──────────┐
│ id  │ name         │ namespace   │ version │ mode    │ pid      │
├─────┼──────────────┼─────────────┼─────────┼─────────┼──────────┤
│ 0   │ claude-code  │ default     │ N/A     │ fork    │ 12345    │
└─────┴──────────────┴─────────────┴─────────┴─────────┴──────────┘
```

Status should show "online".

---

## Essential pm2 Commands

```bash
# See what's running
pm2 status

# View logs (real-time)
pm2 logs claude-code

# View last 100 lines of logs
pm2 logs claude-code --lines 100

# Restart the process
pm2 restart claude-code

# Stop it
pm2 stop claude-code

# Remove it from pm2 entirely
pm2 delete claude-code
```

---

## Starting on Boot

This is the magic bit. You want Claude Code to start automatically when the Pi boots.

```bash
# Generate the startup script
pm2 startup

# This will print a command. Copy and run it.
# It'll look something like:
# sudo env PATH=$PATH:/home/yourusername/.npm-global/bin pm2 startup systemd -u yourusername --hp /home/yourusername

# After running that command, save the current process list
pm2 save
```

Now when your Pi reboots, pm2 will automatically start Claude Code.

**Test it:**
```bash
sudo reboot
```

Wait for it to come back up, SSH in, run `pm2 status`. Claude Code should be running.

---

## Logs and Monitoring

### Where Logs Live

pm2 stores logs in `~/.pm2/logs/`. You'll find:
- `claude-code-out.log` - standard output
- `claude-code-error.log` - errors

### Real-Time Monitoring

```bash
pm2 monit
```

This opens an interactive dashboard showing CPU, memory, and logs. Press `q` to exit.

### Log Rotation (Optional but Recommended)

Logs can get big. Install the log rotation module:

```bash
pm2 install pm2-logrotate
```

Default settings are usually fine. It'll automatically rotate and compress old logs.

---

## Troubleshooting

### Process keeps restarting (restart loop)

Check the error logs:
```bash
pm2 logs claude-code --err --lines 50
```

Common causes:
- Claude Code not properly authenticated (run `claude` manually first)
- Missing environment variables
- Memory issues (Pi running out of RAM)

### "command not found" when pm2 tries to start claude

pm2 might not have your PATH. Specify the full path:
```bash
pm2 delete claude-code
pm2 start ~/.npm-global/bin/claude --name "claude-code"
```

### Process shows "stopped" after boot

The startup script might not have saved properly. Re-run:
```bash
pm2 startup
# Run the command it outputs
pm2 save
```

---

## Environment Variables

If Claude Code needs environment variables (like `DISABLE_AUTOUPDATER`), you can set them in pm2:

```bash
pm2 start claude --name "claude-code" --env DISABLE_AUTOUPDATER=1
```

Or create an ecosystem file for more complex setups (see Advanced section).

---

## Advanced: Ecosystem File

For more control, create an ecosystem file:

```bash
nano ~/claude-ecosystem.config.js
```

```javascript
module.exports = {
  apps: [{
    name: 'claude-code',
    script: '/home/yourusername/.npm-global/bin/claude',
    env: {
      DISABLE_AUTOUPDATER: '1'
    },
    max_memory_restart: '500M',
    restart_delay: 5000
  }]
}
```

Then start with:
```bash
pm2 start ~/claude-ecosystem.config.js
pm2 save
```

---

## What's Next

With pm2 running Claude Code persistently, you've got a foundation. The Pi is now a place Claude can exist, not just visit.

Part 3 covers MCP memory architecture—how to give that persistent Claude actual persistent memory across sessions and platforms.

---

*Last updated: January 13, 2026*
