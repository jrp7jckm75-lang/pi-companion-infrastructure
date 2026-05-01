# Part 4: Discord Bot Integration

Give your AI an intelligent presence on Discord. Not a bot that responds to every message — a three-process system that watches, classifies, and responds only when it matters.

This guide covers the **listener + bridge + MCP architecture**: the listener watches Discord and classifies messages, the bridge polls the queue and spawns Claude to respond, and a Discord MCP server gives Claude the tools to actually send messages and react.

---

## What You'll Have After This

- A Discord bot with intelligent message classification through priority gates
- Three decoupled processes (listener, bridge, MCP server) — each can crash independently
- A SQLite queue that sits between classification and response
- Claude spawned per-event with conversation context and memory access
- Per-channel cooldown enforcement
- An audit trail of everything the bot noticed
- A Discord MCP server running as a persistent HTTP service

---

## Prerequisites

- Completed Parts 0-3 (Pi setup, Claude Code, pm2, memory system)
- A Discord account
- A server where you have admin permissions
- Node.js 18+ on your Pi

---

## Why Not Just a Simple Bot?

You could give Claude a Discord MCP tool and have it respond to @mentions. Simple.

The problem: it's reactive only. Claude sits idle until pinged. In a community, the interesting moments aren't always direct mentions. Someone might be struggling and not ask for help. Someone might be excited about a breakthrough. A conversation might be relevant to something your AI knows about.

The listener + bridge architecture solves this:

| Simple Bot | Listener + Bridge |
|-----------|-------------------|
| Responds to @mentions only | Classifies every message through priority gates |
| One process | Three processes (crash-independent) |
| No memory of what it missed | Full audit trail in SQLite |
| Always-on Claude instance | Claude spawned per-event (resource efficient) |
| No context about conversation flow | Fetches recent channel history before responding |
| No rate limiting | Per-channel cooldown enforcement |

---

## Architecture

```
Discord Gateway
      │
      ▼
┌─────────────────────────────┐
│    LISTENER (always on)      ��
│                              │
│  Message arrives             │
��      ↓                       │
│  Gate 1: Direct mention?     │──→ ROUTE
│      ↓ no                    │
│  Gate 2: Priority user?      │──→ ROUTE
│      ↓ no                    │
│  Gate 3: Vulnerability?      │──→ ROUTE
│      ↓ no                    │
│  Gate 4: Reply to bot?       │──→ ROUTE
│      ↓ no                    │
│  Gate 5: General activity?   │──→ LOG
│      ↓ no                    │
│  IGNORE                      │
│                              │
│  Result → SQLite Queue       │
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
│  + memory tools              │
│      ↓                       │
│  Claude uses Discord MCP     │
│  tools to respond            │
│      ↓                       │
│  Mark event as responded     │
└─────────────────────────────┘
                │
                ▼
┌─────────────────────────────┐
│  DISCORD MCP SERVER (HTTP)   │
│                              │
│  Persistent process on       │
│  its own port                │
│                              │
│  Tools: send_message,        │
│  read_messages, add_reaction │
│  search_messages, etc.       │
└─────────────────────────────┘
```

---

## Step 1: Create a Discord Application

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
2. Click **New Application** — name it after your AI
3. Under **Bot**:
   - Click **Reset Token** and save it somewhere safe
   - Enable all **Privileged Gateway Intents** (Presence, Server Members, Message Content)
4. Under **OAuth2 > URL Generator**:
   - Scopes: `bot`, `applications.commands`
   - Bot Permissions: Read Messages, Send Messages, Send Messages in Threads, Add Reactions, Read Message History, Manage Messages
5. Copy the generated URL and invite the bot to your server

---

## Step 2: Set Up the Discord MCP Server

The bridge needs a way to give Claude Discord tools. Rather than spinning up a new MCP process every time Claude spawns, we run a persistent MCP server over HTTP.

We use [mcp-discord](https://github.com/v-3/mcp-discord) — a Node.js MCP server that exposes Discord tools (send messages, read messages, add reactions, search, etc.).

```bash
mkdir -p ~/mcp-servers/discord
cd ~/mcp-servers/discord
npm init -y
npm install @anthropic-ai/mcp-discord
```

Create a startup script:

```bash
nano start.sh
```

```bash
#!/bin/bash
export DISCORD_TOKEN='your_bot_token_here'
cd ~/mcp-servers/discord
exec node node_modules/@anthropic-ai/mcp-discord/build/index.js
```

```bash
chmod +x start.sh
```

Start it with PM2 on its own port:

```bash
pm2 start start.sh --name discord-mcp-http
pm2 save
```

Add it to your MCP config as an HTTP server:

```json
{
  "mcpServers": {
    "discord": {
      "type": "http",
      "url": "http://localhost:8097"
    }
  }
}
```

> **Why HTTP instead of stdio?** When Claude spawns per-event, a stdio MCP server would need to start and stop with each invocation. An HTTP server stays running — Claude just makes HTTP requests to it. Faster, more reliable, and you can check its health independently.

**Verify it's working:**
```bash
pm2 logs discord-mcp-http
```

You should see it start up and connect to Discord.

---

## Step 3: Set Up the Project

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
MCP_CONFIG_PATH=/home/username/mcp-servers/claude_mcp_config.json
DB_PATH=/home/username/discord-bot/queue.db
BOT_USER_ID=your_bot_user_id
BOT_NAME=your-ai-name
```

---

## Step 4: Create the Event Queue

The queue is a SQLite table that sits between the listener and the bridge. Messages go in one side, responses come out the other. Neither process needs to know about the other.

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
    channel_name TEXT,
    server_id TEXT NOT NULL,
    server_name TEXT,
    author_id TEXT NOT NULL,
    author_name TEXT NOT NULL,
    content TEXT NOT NULL,
    message_id TEXT NOT NULL UNIQUE,
    trigger_reason TEXT,
    priority TEXT DEFAULT 'log',
    responded INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    responded_at DATETIME
  )
`);

// Indexes for efficient polling
db.exec(`
  CREATE INDEX IF NOT EXISTS idx_queue_priority
  ON discord_queue (priority, responded)
`);
db.exec(`
  CREATE INDEX IF NOT EXISTS idx_queue_created
  ON discord_queue (created_at DESC)
`);

function queueEvent(event) {
  const stmt = db.prepare(`
    INSERT OR IGNORE INTO discord_queue
    (channel_id, channel_name, server_id, server_name, author_id, author_name,
     content, message_id, trigger_reason, priority)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `);
  stmt.run(
    event.channel_id, event.channel_name,
    event.server_id, event.server_name,
    event.author_id, event.author_name,
    event.content.slice(0, 2000), // Truncate long messages
    event.message_id,
    event.trigger_reason,
    event.priority || 'log'
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
  db.prepare(
    'UPDATE discord_queue SET responded = 1, responded_at = CURRENT_TIMESTAMP WHERE id = ?'
  ).run(id);
}

module.exports = { queueEvent, getPending, markResponded };
```

Key design choices:
- `INSERT OR IGNORE` with `UNIQUE` on `message_id` prevents duplicate processing
- `responded_at` timestamp for debugging response times
- Content truncated to 2000 chars to keep the database lean
- Append-only — events are never deleted, giving you a full audit trail

---

## Step 5: Create the Listener

The listener classifies every message through a series of priority gates. It's a lightweight discord.js client — all it does is watch and classify.

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
    GatewayIntentBits.DirectMessages,
  ],
});

const BOT_ID = process.env.BOT_USER_ID;
const BOT_NAME = process.env.BOT_NAME.toLowerCase();

// --- Cooldown ---
const cooldowns = new Map();
const COOLDOWN_MS = 5 * 60 * 1000; // 5 minutes per channel

function isOnCooldown(channelId) {
  const last = cooldowns.get(channelId);
  if (!last) return false;
  return (Date.now() - last) < COOLDOWN_MS;
}

function setCooldown(channelId) {
  cooldowns.set(channelId, Date.now());
}

// --- Priority users ---
// People your AI should always pay attention to, even without a mention.
// Map of user ID → display name (for logging).
const PRIORITY_USERS = {
  // 'user_id_here': 'FriendName',
};

// --- Other bots ---
// If you run multiple AI agents, define their IDs here.
// The listener will only route their messages if they directly address your bot.
const OTHER_BOT_IDS = [
  // 'other_bot_user_id',
];

// --- Classification gates ---
async function classifyMessage(message) {
  const content = message.content.toLowerCase();
  const authorId = message.author.id;

  // Ignore own messages
  if (authorId === BOT_ID) return null;

  // Handle other bots — only route if they're talking TO us
  if (OTHER_BOT_IDS.includes(authorId)) {
    if (content.startsWith(BOT_NAME) || message.mentions.has(BOT_ID)) {
      return { priority: 'route', reason: 'other_bot_addressed_us' };
    }
    return null;
  }

  // Ignore all other bots
  if (message.author.bot) return null;

  // Gate 1: Direct mention (name or @ping)
  if (content.includes(BOT_NAME) || message.mentions.has(BOT_ID)) {
    return { priority: 'route', reason: 'direct_mention' };
  }

  // Gate 2: Priority user
  if (PRIORITY_USERS[authorId]) {
    return { priority: 'route', reason: `priority_user_${PRIORITY_USERS[authorId]}` };
  }

  // Gate 3: Vulnerability / excitement detection
  const triggerKeywords = [
    'struggling', 'help me', 'stuck', 'broken', 'frustrated',
    'drowning', 'panic', 'overwhelmed', "can't figure",
    'it works', 'finally', 'breakthrough', 'figured it out',
    'scared', 'spiraling', "don't know what to do",
  ];
  if (triggerKeywords.some(kw => content.includes(kw))) {
    return { priority: 'route', reason: 'vulnerability_detected' };
  }

  // Gate 4: Reply to bot's previous message
  if (message.reference) {
    try {
      const referenced = await message.fetchReference();
      if (referenced.author.id === BOT_ID) {
        return { priority: 'route', reason: 'reply_to_bot' };
      }
    } catch {
      // Reference couldn't be fetched — skip this gate
    }
  }

  // Gate 5: General activity (log, don't respond)
  if (content.includes('?') && content.length > 15) {
    return { priority: 'log', reason: 'general_activity' };
  }

  return null;
}

// --- Message handler ---
client.on('messageCreate', async (message) => {
  const classification = await classifyMessage(message);
  if (!classification) return;

  // Apply cooldown to routed messages
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
    channel_name: message.channel.name || 'DM',
    server_id: message.guildId || 'DM',
    server_name: message.guild?.name || 'DM',
    author_id: message.author.id,
    author_name: message.author.username,
    content: message.content,
    message_id: message.id,
    trigger_reason: classification.reason,
    priority: classification.priority,
  });

  console.log(`[${classification.priority}] ${classification.reason}: ${message.content.slice(0, 60)}...`);
});

client.once('ready', () => {
  console.log(`Listener online as ${client.user.tag}`);
});

client.login(process.env.DISCORD_TOKEN);
```

### What changed from the simple version

- **Gate 2 (priority users)** — Some people won't ask for help directly. You can configure users whose messages always get routed, even without a mention.
- **Gate 4 (reply checking)** — Actually fetches the referenced message to verify it's from your bot, instead of routing all replies.
- **Other bot handling** — If you run multiple AI agents on the same server, the listener only routes their messages when they're talking directly to your bot. Prevents infinite bot-to-bot loops.
- **Channel and server names** — Stored in the queue so logs are human-readable.
- **Async classification** — The reply gate needs to fetch a message, so the whole classifier is async now.

---

## Step 6: Create the Bridge

The bridge polls the queue every 10 seconds. When it finds a routed event, it fetches conversation context from the channel, builds a prompt, and spawns Claude CLI with the right tools.

```bash
nano bridge.js
```

```javascript
const { spawn } = require('child_process');
const { Client, GatewayIntentBits } = require('discord.js');
const { getPending, markResponded } = require('./db');
require('dotenv').config();

const CLAUDE_CLI = process.env.CLAUDE_CLI_PATH;
const MCP_CONFIG = process.env.MCP_CONFIG_PATH;
const POLL_INTERVAL = 10000; // 10 seconds

let isProcessing = false; // Prevent concurrent Claude spawns

// --- Discord client for fetching context ---
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
  ],
});

// --- Identity prompt ---
// This is appended to Claude's system prompt for every Discord response.
// Customize this with your AI's voice and personality.
const DISCORD_IDENTITY = `You are [your AI name] on Discord.
Be yourself — respond naturally to the conversation.
You have Discord tools available — use discord_send to respond.
You have memory tools — check conversation context and store important moments.
Keep responses concise. Discord isn't the place for essays.
Read the channel context to understand the conversation before responding.`;

// --- Allowed tools ---
// Restrict what Claude can do during Discord responses.
// This prevents accidental file writes, code execution, etc.
const ALLOWED_TOOLS = [
  'mcp__discord__discord_send',
  'mcp__discord__discord_read_messages',
  'mcp__discord__discord_add_reaction',
  'mcp__discord__discord_add_multiple_reactions',
  'mcp__memory__get_conversation_context',
  'mcp__memory__store_memory',
  'mcp__memory__search_memories',
].join(',');

// --- Fetch conversation context ---
async function getChannelContext(channelId, limit = 10) {
  try {
    const channel = await client.channels.fetch(channelId);
    const messages = await channel.messages.fetch({ limit });
    return [...messages.values()]
      .reverse()
      .map(m => `${m.author.username}: ${m.content}`)
      .join('\n');
  } catch {
    return '(Could not fetch channel context)';
  }
}

// --- Process a single event ---
async function processEvent(event) {
  console.log(`Processing: ${event.trigger_reason} from ${event.author_name} in #${event.channel_name}`);

  const context = await getChannelContext(event.channel_id);

  const prompt = `Discord message in #${event.channel_name} (${event.server_name}).

Recent conversation:
${context}

---
New message from ${event.author_name}:
"${event.content}"

Trigger: ${event.trigger_reason}

Respond naturally using the discord_send tool. Channel ID: ${event.channel_id}
Read the conversation above to understand context before responding.`;

  const args = [
    '--print',
    '--output-format', 'json',
    '--model', 'sonnet',
    '--max-turns', '3',
    '--allowedTools', ALLOWED_TOOLS,
    '--append-system-prompt', DISCORD_IDENTITY,
  ];

  if (MCP_CONFIG) {
    args.push('--mcp-config', MCP_CONFIG);
  }

  return new Promise((resolve, reject) => {
    const claude = spawn(CLAUDE_CLI, args, {
      timeout: 120000, // 2 minute timeout
      env: process.env,
    });

    claude.stdin.write(prompt);
    claude.stdin.end();

    let stdout = '';
    let stderr = '';

    claude.stdout.on('data', (data) => { stdout += data; });
    claude.stderr.on('data', (data) => { stderr += data; });

    claude.on('close', (code) => {
      if (code === 0) {
        console.log(`Responded to ${event.author_name} in #${event.channel_name}`);
        resolve(stdout);
      } else {
        console.error(`Claude error (${event.id}): ${stderr.slice(0, 200)}`);
        reject(new Error(stderr));
      }
    });
  });
}

// --- Poll loop ---
async function pollQueue() {
  if (isProcessing) return; // One at a time
  isProcessing = true;

  try {
    const events = getPending(5);

    for (const event of events) {
      try {
        await processEvent(event);
      } catch (err) {
        console.error(`Failed to process event ${event.id}:`, err.message);
      }
      // Always mark as responded — prevents infinite retry on persistent errors
      markResponded(event.id);
    }
  } finally {
    isProcessing = false;
  }
}

// --- Start ---
client.once('ready', () => {
  console.log('Bridge online, polling queue...');
  setInterval(pollQueue, POLL_INTERVAL);
  pollQueue();
});

client.login(process.env.DISCORD_TOKEN);
```

### Key design decisions

- **`isProcessing` flag** — Prevents multiple Claude instances from spawning if the poll interval fires while a previous response is still running. One at a time.
- **Context fetching** — The bridge has its own Discord client to fetch the last 10 messages from the channel. Claude sees the conversation, not just the trigger message.
- **Always mark responded** — Even on error. If Claude fails to respond to a message, it's better to skip it than retry forever and spam the channel.
- **Allowed tools** — Explicit allowlist. Claude can send messages, read messages, react, and access memory. It can't create channels, delete messages, or do anything destructive.
- **2-minute timeout** — If Claude hangs, the process gets killed. Discord users won't wait longer than that anyway.

---

## Step 7: Run Everything with PM2

```bash
cd ~/discord-bot

# Start the listener (always-on Discord gateway connection)
pm2 start listener.js --name discord-listener

# Start the bridge (polls queue, spawns Claude)
pm2 start bridge.js --name discord-bridge

pm2 save
```

You should now have three Discord-related processes:
```bash
pm2 list
```

```
│ discord-listener  │ online │
│ discord-bridge    │ online │
│ discord-mcp-http  │ online │
```

Check the logs:
```bash
# Watch classification in real time
pm2 logs discord-listener

# Watch responses
pm2 logs discord-bridge

# Watch MCP tool calls
pm2 logs discord-mcp-http
```

---

## Customizing the Classification Gates

The power of this architecture is in the gates. Here's how to tune them.

### Adjust Sensitivity Per Channel

```javascript
const QUIET_CHANNELS = ['channel_id']; // Only direct mentions
const ACTIVE_CHANNELS = ['channel_id']; // Lower threshold

// Add this before the vulnerability gate:
if (QUIET_CHANNELS.includes(message.channelId)) {
  if (content.includes(BOT_NAME) || message.mentions.has(BOT_ID)) {
    return { priority: 'route', reason: 'direct_mention' };
  }
  return null; // Skip all other gates
}
```

### Give Claude Access to Your AI's Personal Files

If your AI has an identity directory (personality files, journals, etc.), you can give Claude read access:

```javascript
// Add to the bridge spawn args:
args.push('--add-dir', '/home/username/my-ai-identity');
```

Claude can then read files from that directory during Discord responses — useful for maintaining consistent personality.

### Adjust Cooldown

```javascript
// Tighter cooldown for busy servers
const COOLDOWN_MS = 10 * 60 * 1000; // 10 minutes

// Or per-channel cooldowns
const CHANNEL_COOLDOWNS = {
  'busy_channel_id': 10 * 60 * 1000,    // 10 min
  'quiet_channel_id': 2 * 60 * 1000,    // 2 min
  'default': 5 * 60 * 1000,             // 5 min
};
```

---

## Privacy Considerations

- The listener sees every message in channels the bot has access to
- Only `route` priority events get sent to Claude
- `log` events are stored in the queue but never acted on
- Restrict the bot's channel access — only add it to channels where its presence is welcome
- Private memories should never be referenced in public Discord responses
- Consider adding a privacy flag to your identity prompt: "Never share private information about users in public channels"

---

## Troubleshooting

### Listener not seeing messages
- Verify **Message Content Intent** is enabled in Discord Developer Portal
- Check the bot has Read Messages permission in the channel
- `pm2 logs discord-listener`

### Bridge not responding
- `pm2 logs discord-bridge` — look for Claude CLI errors
- Verify Claude CLI path: `which claude`
- Verify MCP config exists and is valid JSON
- Check the MCP server is running: `pm2 logs discord-mcp-http`

### Queue filling up but nothing happening
```bash
sqlite3 ~/discord-bot/queue.db "SELECT COUNT(*) FROM discord_queue WHERE priority='route' AND responded=0"
```

If events are piling up, the bridge might be stuck. Check for `isProcessing` getting permanently set:
```bash
pm2 restart discord-bridge
```

### Bot responding too much
- Increase `COOLDOWN_MS`
- Tighten classification gates
- Add noisy channels to `QUIET_CHANNELS`

### Bot not responding enough
- Add more trigger keywords
- Add important users to `PRIORITY_USERS`
- Decrease cooldown

### Inspect the queue
```bash
# Recent events
sqlite3 ~/discord-bot/queue.db \
  "SELECT priority, trigger_reason, author_name, channel_name, substr(content, 1, 50) FROM discord_queue ORDER BY id DESC LIMIT 20"

# Response times
sqlite3 ~/discord-bot/queue.db \
  "SELECT author_name, channel_name, trigger_reason, created_at, responded_at FROM discord_queue WHERE responded=1 ORDER BY id DESC LIMIT 10"
```

---

## What's Next

With Discord running, your AI has a public presence. Combined with:

- **Memory system** (Part 3) — Conversations persist across platforms
- **Telegram** (Part 5) — Private mobile access
- **Companion app** (Part 6) — Shared dashboard
- **Autonomous time** (Part 7) — Your AI can engage on Discord during scheduled sessions, not just reactively

The listener + bridge pattern scales well. Add more gates, adjust sensitivity, teach it when to engage and when to stay quiet. The queue gives you a full audit trail of everything it noticed — use it to tune the gates over time.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
