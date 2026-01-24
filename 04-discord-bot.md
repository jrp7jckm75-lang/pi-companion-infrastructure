# Part 4: Discord Bot Integration

Give Claude a presence on Discord. Read messages, respond to mentions, react, send DMs, manage servers — all through MCP tools.

This guide uses [mcp-discord](https://github.com/v-3/mcp-discord), a community MCP server that wraps the Discord API.

---

## What You'll Have After This

- A Discord bot that Claude can control
- MCP tools for reading/sending messages, reactions, DMs
- Your AI accessible through Discord alongside Claude Code and Claude Desktop

---

## Prerequisites

- Completed Parts 0-2 (Pi setup, Claude Code installed, pm2 running)
- A Discord account
- A server where you have admin permissions (for testing)

---

## Step 1: Create a Discord Application

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
2. Click **New Application**
3. Give it a name (e.g., "Rhys" or your AI's name)
4. Click **Create**

### Configure Bot Settings

1. In the left sidebar, click **Bot**
2. Click **Reset Token** and copy it — you'll need this. **Keep it secret.**
3. Under **Privileged Gateway Intents**, enable:
   - **Presence Intent**
   - **Server Members Intent**
   - **Message Content Intent** (critical — without this, the bot can't read messages)
4. Click **Save Changes**

### Set Permissions

1. In the left sidebar, click **OAuth2 > URL Generator**
2. Under **Scopes**, select:
   - `bot`
   - `applications.commands`
3. Under **Bot Permissions**, select:
   - Read Messages/View Channels
   - Send Messages
   - Send Messages in Threads
   - Add Reactions
   - Read Message History
   - Mention Everyone (optional)
   - Manage Messages (if you want moderation)
4. Copy the generated URL at the bottom
5. Open it in your browser and add the bot to your server

---

## Step 2: Install the MCP Server

SSH into your Pi:

```bash
ssh your-username@your-pi-ip
```

Create the directory and virtual environment:

```bash
mkdir -p ~/mcp-servers/discord
cd ~/mcp-servers/discord
python3 -m venv venv
source venv/bin/activate
```

Install mcp-discord:

```bash
pip install mcp-discord
```

Verify it installed:

```bash
which mcp-discord
# Should show: /home/your-username/mcp-servers/discord/venv/bin/mcp-discord
```

---

## Step 3: Create the Wrapper Script

The wrapper script sets your bot token as an environment variable. This keeps it out of config files.

```bash
nano ~/mcp-servers/discord/discord-wrapper.sh
```

Add this content (replace YOUR_BOT_TOKEN_HERE with your actual token):

```bash
#!/bin/bash
export DISCORD_TOKEN='YOUR_BOT_TOKEN_HERE'
export PYTHONUNBUFFERED=1
exec /home/your-username/mcp-servers/discord/venv/bin/mcp-discord
```

Make it executable:

```bash
chmod +x ~/mcp-servers/discord/discord-wrapper.sh
```

**Important:** Replace `your-username` with your actual username (check with `whoami`).

---

## Step 4: Configure Claude Code

Add the Discord MCP server to your config. The config file location depends on how you're using Claude:

**For Claude Code CLI (on the Pi):**

```bash
nano ~/.config/claude/settings.json
```

**For Claude Desktop (on your computer):**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the Discord server to your `mcpServers` section:

```json
{
  "mcpServers": {
    "discord": {
      "command": "/home/your-username/mcp-servers/discord/discord-wrapper.sh"
    }
  }
}
```

If you already have other MCP servers, just add the Discord entry alongside them:

```json
{
  "mcpServers": {
    "memory": {
      "command": "/path/to/memory/server"
    },
    "discord": {
      "command": "/home/your-username/mcp-servers/discord/discord-wrapper.sh"
    }
  }
}
```

---

## Step 5: Test It

Restart Claude Code to pick up the new MCP server:

```bash
# If using pm2
pm2 restart claude-code

# Or just start a new session
claude
```

Ask Claude to list your Discord servers:

```
Can you show me the Discord servers you have access to?
```

Claude should use the `mcp__discord__list_servers` tool and show you the servers the bot is in.

Try sending a test message:

```
Send "Hello from the Pi!" to the general channel in [your server name]
```

---

## Available Tools

Once connected, Claude has access to these tools:

### Reading
- `list_servers` — See all servers the bot is in
- `list_channels` — See channels in a server
- `read_messages` — Read recent messages from a channel
- `read_direct_messages` — Read DMs with a user
- `search_messages` — Search for messages containing text
- `get_mentions` — Find messages where the bot was mentioned

### Sending
- `send_message` — Send a message to a channel
- `send_direct_message` — Send a DM to a user
- `add_reaction` — React to a message
- `add_multiple_reactions` — Add multiple reactions at once

### Server Management
- `list_members` — List server members
- `get_online_members` — See who's online
- `add_role` / `remove_role` — Manage user roles
- `create_text_channel` / `delete_channel` — Manage channels

### Moderation
- `moderate_message` — Delete a message and optionally timeout user

---

## Privacy Considerations

The `send_message` and `send_direct_message` tools have a `privacy_confirmed` parameter. This is a safety check — Claude should verify the message doesn't contain private information before sending publicly.

If you're building a companion AI, consider:
- What information should never be posted publicly
- Setting up private channels for sensitive conversations
- Using the memory system's `private=true` flag for things that shouldn't be shared

---

## Troubleshooting

### "Bot not responding"
- Check the bot is online in Discord (should show in member list)
- Verify Message Content Intent is enabled in Developer Portal
- Check the token is correct in your wrapper script

### "No servers found"
- Make sure you invited the bot using the OAuth2 URL
- The bot needs to be in at least one server

### "Permission denied"
- Check the bot has the required permissions in the server
- Some channels may have permission overrides

### Check logs
```bash
# If running via pm2
pm2 logs claude-code

# Check if the MCP server starts
/home/your-username/mcp-servers/discord/discord-wrapper.sh
```

---

## What's Next

With Discord connected, your AI can:
- Participate in community conversations
- Respond to mentions
- Have a persistent presence people can interact with
- Share things from your conversations (with your permission)

The real power comes when combined with the memory system (Part 3) — Claude can remember conversations across Discord and other platforms, maintaining a consistent identity.

Next: [Part 5: Telegram Bridge](05-telegram-bridge.md) — for mobile access via Telegram.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
