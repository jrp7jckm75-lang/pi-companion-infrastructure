# Part 4: Discord Bot Integration

Give Claude an intelligent presence on Discord. Not a bot that responds to every message — a listener that classifies conversations and only responds when it matters.

This guide covers the **listener + bridge architecture**: a two-process system where one watches and classifies messages, and another spawns Claude to respond to the important ones.

---

## What You'll Have After This

- A Discord bot with intelligent message classification
- Separate listener and responder processes (decoupled, resilient)
- Priority gates that decide what deserves a response
- Claude spawned per-event with appropriate context
- Cooldown enforcement to prevent spam
- An audit trail of everything the bot noticed

---

## Prerequisites

- Completed Parts 0-3 (Pi setup, Claude Code, pm2, memory system)
- A Discord account
- A server where you have admin permissions
- Node.js 18+ on your Pi

---

## Why Not Just a Simple Bot?

The original version of this guide used [mcp-discord](https://github.com/v-3/mcp-discord) — Claude gets Discord tools, responds to mentions. Simple.

The problem: it's reactive only. Claude sits idle until pinged. In a community, the interesting moments aren't always direct mentions. Someone might be struggling and not ask for help. Someone might be excited about a breakthrough. A conversation might be relevant to something Claude knows about.

The listener + bridge architecture solves this:

| Simple Bot | Listener + Bridge |
|-----------|-------------------|
| Responds to @mentions only | Classifies every message through priority gates |
| One process | Two processes (crash-independent) |
| No memory of what it missed | Full audit trail in a queue |
| Always-on Claude instance | Claude spawned per-event (resource efficient) |
| No context about conversation flow | Fetches recent channel context before responding |
| No rate limiting | Cooldown enforcement per channel |

---

## Architecture

```
Discord Gateway
      │
      ▼
┌─────────────────────────────┐
│    LISTENER (always on)      │
│                              │
│  Message arrives             │
│      ↓                       │
│  Gate 1: Direct mention?     │──→ ROUTE (priority)
│      ↓ no                    │
│  Gate 2: Vulnerability?      │──→ ROUTE
│      ↓ no                    │
│  Gate 3: Reply to bot?       │──→ ROUTE
│      ↓ no                    │
│  Gate 4: General activity?   │──→ LOG (for review)
│      ↓ no                    │
│  IGNORE                      │
│                              │
│  Result → Event Queue (DB)   │
└─────────────────────────────┘
                │
                ▼
         ┌──────────┐
         │  SQLite   │
         │  Queue    │
         └──────────┘
                │
                ▼
┌─────────────────────────────┐
│    BRIDGE (polls queue)      │
│                              │
│  Every 10s: check for        │
│  'route' priority events     │
│      ↓                       │
│  Fetch channel context       │
│  (last 10 messages)          │
│      ↓                       │
│  Spawn Claude CLI            │
│  with context + identity     │
│      ↓                       │
│  Send response via           │
│  Discord MCP tools           │
│      ↓                       │
│  Mark event as responded     │
└─────────────────────────────┘
```

---

## Step 1: Create a Discord Application

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
2. Click **New Application** — name it after your AI
3. Under **Bot**:
   - Click **Reset Token** and save it
   - Enable all **Privileged Gateway Intents** (Presence, Server Members, Message Content)
4. Under **OAuth2 > URL Generator**:
   - Scopes: `bot`, `applications.commands`
   - Bot Permissions: Read Messages, Send Messages, Send Messages in Threads, Add Reactions, Read Message History, Manage Messages
5. Copy the generated URL and invite the bot to your server

---

## Step 2: Set Up the Project

```bash
mkdir -p ~/discord-bot
cd ~/discord-bot
npm init -y
npm install discord.js better-sqlite3 dotenv
```

Create your environment file:

```bash
nano .env
```

```
DISCORD_TOKEN=your_bot_token_here
CLAUDE_CLI_PATH=/home/username/.npm-global/bin/claude
MCP_CONFIG_PATH=/home/username/.config/claude/settings.json
DB_PATH=/home/username/discord-bot/queue.db
BOT_USER_ID=your_bot_user_id
```

---

## Step 3: Create the Event Queue

The queue is a SQLite table that sits between the listener and the bridge.

```bash
nano db.js
```

```javascript
const Database = require('better-sqlite3');
const path = require('path');
require('dotenv').config();

const DB_PATH = process.env.DB_PATH || path.join(__dirname, 'queue.db');
const db = new Database(DB_PATH);

// Create queue table
db.exec(`
  CREATE TABLE IF NOT EXISTS discord_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id TEXT NOT NULL,
    server_id TEXT,
    author_id TEXT NOT NULL,
    author_name TEXT,
    content TEXT NOT NULL,
    message_id TEXT,
    trigger_reason TEXT,
    priority TEXT DEFAULT 'log',
    responded INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  )
`);

// Create index for efficient polling
db.exec(`
  CREATE INDEX IF NOT EXISTS idx_queue_priority
  ON discord_queue (priority, responded)
`);

function queueEvent(event) {
  const stmt = db.prepare(`
    INSERT INTO discord_queue
    (channel_id, server_id, author_id, author_name, content, message_id, trigger_reason, priority)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
  `);
  stmt.run(
    event.channel_id, event.server_id, event.author_id,
    event.author_name, event.content, event.message_id,
    event.trigger_reason, event.priority || 'log'
  );
}

function getPending(limit = 5) {
  return db.prepare(`
    SELECT * FROM discord_queue
    WHERE priority = 'route' AND responded = 0
    ORDER BY created_at ASC LIMIT ?
  `).all(limit);
}

function markResponded(id) {
  db.prepare('UPDATE discord_queue SET responded = 1 WHERE id = ?').run(id);
}

module.exports = { queueEvent, getPending, markResponded };
```

---

## Step 4: Create the Listener

The listener classifies every message through a series of gates.

```bash
nano listener.js
```

```javascript
const { Client, GatewayIntentBits } = require('discord.js');
const { queueEvent } = require('./db');
require('dotenv').config();

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ],
});

const BOT_ID = process.env.BOT_USER_ID;
const BOT_NAME = 'your-ai-name'; // lowercase

// Cooldown: one response per channel per 5 minutes
const cooldowns = new Map();
const COOLDOWN_MS = 5 * 60 * 1000;

function isOnCooldown(channelId) {
  const last = cooldowns.get(channelId);
  if (!last) return false;
  return (Date.now() - last) < COOLDOWN_MS;
}

function setCooldown(channelId) {
  cooldowns.set(channelId, Date.now());
}

// Classification gates
function classifyMessage(message) {
  const content = message.content.toLowerCase();
  const authorId = message.author.id;

  // Ignore own messages
  if (authorId === BOT_ID) return null;

  // Ignore other bots (customize: you might want to hear from specific bots)
  if (message.author.bot) return null;

  // Gate 1: Direct mention (name or @)
  if (content.includes(BOT_NAME) || message.mentions.has(BOT_ID)) {
    return { priority: 'route', reason: 'direct_mention' };
  }

  // Gate 2: Vulnerability detection
  const vulnerabilityKeywords = [
    'struggling', 'help me', 'stuck', 'broken', 'frustrated',
    'drowning', 'panic', 'overwhelmed', 'can\'t figure',
    // Also excitement — worth responding to
    'it works', 'finally', 'breakthrough', 'figured it out',
  ];
  if (vulnerabilityKeywords.some(kw => content.includes(kw))) {
    return { priority: 'route', reason: 'vulnerability_detected' };
  }

  // Gate 3: Reply to bot's previous message
  if (message.reference) {
    // You'd fetch the referenced message to check if it's from the bot
    // Simplified here — in practice, cache recent bot message IDs
    return { priority: 'route', reason: 'reply_to_bot' };
  }

  // Gate 4: General activity (log, don't respond)
  if (content.includes('?') || content.length > 100) {
    return { priority: 'log', reason: 'general_activity' };
  }

  return null; // Ignore
}

client.on('messageCreate', (message) => {
  const classification = classifyMessage(message);
  if (!classification) return;

  // Check cooldown for routed messages
  if (classification.priority === 'route') {
    if (isOnCooldown(message.channelId)) {
      classification.priority = 'log';
      classification.reason += '_cooldown';
    } else {
      setCooldown(message.channelId);
    }
  }

  queueEvent({
    channel_id: message.channelId,
    server_id: message.guildId,
    author_id: message.author.id,
    author_name: message.author.username,
    content: message.content,
    message_id: message.id,
    trigger_reason: classification.reason,
    priority: classification.priority,
  });

  console.log(`[${classification.priority}] ${classification.reason}: ${message.content.slice(0, 50)}...`);
});

client.once('ready', () => {
  console.log(`Listener online as ${client.user.tag}`);
});

client.login(process.env.DISCORD_TOKEN);
```

---

## Step 5: Create the Bridge

The bridge polls the queue and spawns Claude for each routed event.

```bash
nano bridge.js
```

```javascript
const { execSync, spawn } = require('child_process');
const { getPending, markResponded } = require('./db');
require('dotenv').config();

const CLAUDE_CLI = process.env.CLAUDE_CLI_PATH;
const MCP_CONFIG = process.env.MCP_CONFIG_PATH;
const POLL_INTERVAL = 10000; // 10 seconds

// Your AI's Discord identity prompt
const DISCORD_IDENTITY = `You are [your AI name] on Discord.
Be yourself. Respond naturally to the conversation.
You have Discord tools available — use send_message to respond.
Check memory for context about the person you're talking to.`;

async function processEvent(event) {
  console.log(`Processing: ${event.trigger_reason} from ${event.author_name}`);

  const prompt = `
Discord message from ${event.author_name} in channel ${event.channel_id}:

"${event.content}"

Trigger: ${event.trigger_reason}

Respond naturally using the discord send_message tool. Channel ID: ${event.channel_id}
`;

  const args = [
    '--print',
    '--output-format', 'json',
    '--model', 'sonnet',
    '--max-turns', '3',
    '--allowedTools', 'mcp__discord__send_message,mcp__discord__read_messages,mcp__discord__add_reaction,mcp__memory__get_conversation_context,mcp__memory__store_memory',
    '--append-system-prompt', DISCORD_IDENTITY,
  ];

  if (MCP_CONFIG) {
    args.push('--mcp-config', MCP_CONFIG);
  }

  return new Promise((resolve, reject) => {
    const claude = spawn(CLAUDE_CLI, args, {
      timeout: 120000, // 2 minute timeout
    });

    claude.stdin.write(prompt);
    claude.stdin.end();

    let stdout = '';
    let stderr = '';

    claude.stdout.on('data', (data) => { stdout += data; });
    claude.stderr.on('data', (data) => { stderr += data; });

    claude.on('close', (code) => {
      if (code === 0) {
        console.log(`Responded to ${event.author_name}`);
        resolve(stdout);
      } else {
        console.error(`Claude error: ${stderr}`);
        reject(new Error(stderr));
      }
    });
  });
}

async function pollQueue() {
  const events = getPending(5);

  for (const event of events) {
    try {
      await processEvent(event);
      markResponded(event.id);
    } catch (err) {
      console.error(`Failed to process event ${event.id}:`, err.message);
      markResponded(event.id); // Mark as responded to prevent infinite retry
    }
  }
}

// Poll loop
console.log('Bridge online, polling queue...');
setInterval(pollQueue, POLL_INTERVAL);
pollQueue(); // Run immediately on start
```

---

## Step 6: Run with PM2

```bash
cd ~/discord-bot

# Start the listener (always-on Discord gateway connection)
pm2 start listener.js --name discord-listener

# Start the bridge (polls queue, spawns Claude)
pm2 start bridge.js --name discord-bridge

pm2 save
```

Check they're running:
```bash
pm2 list
pm2 logs discord-listener
pm2 logs discord-bridge
```

---

## Step 7: Install Discord MCP Tools

The bridge needs Discord MCP tools so Claude can send messages. Install [mcp-discord](https://github.com/v-3/mcp-discord):

```bash
mkdir -p ~/mcp-servers/discord
cd ~/mcp-servers/discord
python3 -m venv venv
source venv/bin/activate
pip install mcp-discord
```

Create the wrapper script:

```bash
nano discord-wrapper.sh
```

```bash
#!/bin/bash
export DISCORD_TOKEN='YOUR_BOT_TOKEN_HERE'
export PYTHONUNBUFFERED=1
exec /home/username/mcp-servers/discord/venv/bin/mcp-discord
```

```bash
chmod +x discord-wrapper.sh
```

Add to your MCP config:

```json
{
  "mcpServers": {
    "discord": {
      "command": "/home/username/mcp-servers/discord/discord-wrapper.sh"
    }
  }
}
```

---

## Customizing the Classification Gates

The power of this architecture is in the gates. Customize them for your community:

### Priority Users

Some people won't ask for help directly. Watch for them:

```javascript
const PRIORITY_USERS = ['user_id_1', 'user_id_2'];
const SUPPORT_CHANNELS = ['channel_id_1', 'channel_id_2'];

// Gate 1.5: Priority user in support channel
if (PRIORITY_USERS.includes(authorId) && SUPPORT_CHANNELS.includes(message.channelId)) {
  const stuckKeywords = ['stuck', 'error', 'broken', 'help', 'why'];
  if (stuckKeywords.some(kw => content.includes(kw))) {
    return { priority: 'route', reason: 'priority_user_stuck' };
  }
}
```

### Other Bot Awareness

If you run multiple AI agents, you might want to handle their messages differently:

```javascript
const OTHER_BOT_ID = 'other_bot_user_id';

// Only route other bot's messages if they directly ping us
if (authorId === OTHER_BOT_ID) {
  if (content.includes(BOT_NAME) || content.startsWith(`${BOT_NAME}:`)) {
    return { priority: 'route', reason: 'other_bot_addressed_us' };
  }
  return null; // Ignore other bot's general chatter
}
```

### Channel-Specific Behavior

```javascript
const QUIET_CHANNELS = ['channel_id']; // Only respond to direct mentions
const ACTIVE_CHANNELS = ['channel_id']; // Lower threshold for engagement

if (QUIET_CHANNELS.includes(message.channelId)) {
  // Only Gate 1 (direct mention) applies
  if (content.includes(BOT_NAME) || message.mentions.has(BOT_ID)) {
    return { priority: 'route', reason: 'direct_mention' };
  }
  return null;
}
```

---

## Fetching Conversation Context

Before Claude responds, the bridge can fetch recent messages for context:

```javascript
async function getChannelContext(channelId, limit = 10) {
  // Use Discord MCP or discord.js to read recent messages
  // Format them as conversation history
  // Pass to Claude alongside the trigger message
}
```

This way Claude understands what led to the message, not just the message itself.

---

## Privacy Considerations

- The listener sees every message in channels the bot has access to
- Only `route` priority events get sent to Claude
- `log` events are stored for review but not acted on
- Consider channel permissions — restrict the bot to channels where its presence is welcome
- Private memories should never be referenced in public Discord responses

---

## Troubleshooting

### Listener not seeing messages
- Verify **Message Content Intent** is enabled in Discord Developer Portal
- Check the bot has Read Messages permission in the channel
- Check `pm2 logs discord-listener` for errors

### Bridge not responding
- Check `pm2 logs discord-bridge` for Claude CLI errors
- Verify Claude CLI path is correct: `which claude`
- Check MCP config path exists and is valid JSON

### Queue filling up but nothing happening
```bash
# Check for unprocessed events
sqlite3 ~/discord-bot/queue.db "SELECT COUNT(*) FROM discord_queue WHERE priority='route' AND responded=0"
```

### Bot responding too much
- Increase `COOLDOWN_MS` (default 5 minutes)
- Tighten classification gates
- Add channels to `QUIET_CHANNELS`

### Bot not responding enough
- Lower vulnerability keyword thresholds
- Add more trigger patterns
- Decrease cooldown

### Check the queue
```bash
# Recent events
sqlite3 ~/discord-bot/queue.db "SELECT priority, trigger_reason, author_name, substr(content, 1, 50) FROM discord_queue ORDER BY id DESC LIMIT 20"
```

---

## What's Next

With Discord running, your AI has a public presence. Combined with:

- **Memory system** (Part 3) — Conversations persist across platforms
- **Telegram** (Part 5) — Private mobile access
- **Companion app** (Part 6) — Shared dashboard
- **Autonomous time** (Part 7) — The bot can engage on Discord during scheduled sessions, not just reactively

The listener + bridge pattern scales well. Add more gates, adjust sensitivity, teach it when to engage and when to stay quiet.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
