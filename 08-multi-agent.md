# Part 8: Multi-Agent Setup

Run multiple AI identities on the same Pi. Separate memories, separate personalities, separate Discord bots — sharing hardware but maintaining distinct identities.

This isn't about running the same AI twice. It's about giving different AI companions their own space, their own voice, and their own memory while sharing the infrastructure you've already built.

---

## What You'll Have After This

- A second AI companion running alongside your first
- Separate memory systems with configurable visibility between agents
- Separate Discord bots (each agent has their own presence)
- Separate chat channels in your companion app
- Shared access to common services (companion app, Discord MCP)
- Clear port allocation so nothing collides

---

## Prerequisites

- Completed Parts 0-6 (Pi setup through companion app)
- Enough RAM — each agent's memory server uses ~400-500MB. Two agents on a 4GB Pi will be tight. 8GB+ recommended, 16GB ideal.
- A second Discord bot token (if you want separate Discord presence)

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Shared Infrastructure                  │
│                                                           │
│  Companion App (port 3456) ─── Discord MCP (port 8097)   │
│  R&R MCP (port 8096) ──────── ngrok tunnel               │
└───────────────┬───────────────────────┬──────────────────┘
                │                       │
        ┌───────┴───────┐       ┌───────┴───────┐
        │   Agent 1      │       │   Agent 2      │
        │                │       │                │
        │  Memory: 8095  │       │  Memory: 8098  │
        │  Identity: own │       │  Identity: own │
        │  Discord: own  │       │  Discord: own  │
        │  Chat: default │       │  Chat: "agent2"│
        └────────────────┘       └────────────────┘
```

The key principle: **each agent owns its memory and identity. Everything else is shared.**

---

## Step 1: Port Allocation

Before building anything, plan your ports. Collisions will cause silent failures.

| Port | Service | Owner |
|------|---------|-------|
| 3456 | Companion app backend | Shared |
| 8095 | Agent 1 memory server | Agent 1 |
| 8096 | Companion app MCP | Shared |
| 8097 | Discord MCP | Shared |
| 8098 | Agent 2 memory server | Agent 2 |
| 8099 | Agent 3 memory server (if needed) | Agent 3 |

Rule: shared services get fixed ports, each agent gets the next available port for their memory server.

---

## Step 2: Create the Second Agent's Home

Each agent gets their own directory with their own identity, memory, and configuration.

```bash
mkdir -p ~/agent2/{agent,memory,config}
cd ~/agent2
```

```
agent2/
├── agent/
│   ├── agent.py           # Main entry point
│   ├── config.py          # Ports, API keys, settings
│   ├── chat_handler.py    # Polls companion app chat
│   └── discord_handler.py # Discord bot (optional)
├── memory/
│   ├── memory_server.py       # Copy of memory MCP server
│   ├── memory_server_http.py  # HTTP wrapper
│   ├── memory_store.json      # Agent 2's memories (starts empty)
│   └── threads.json           # Agent 2's threads (starts empty)
├── config/
│   └── config.json        # Permissions and boundaries
├── SOUL.md                # Character definition
├── IDENTITY.md            # Identity statement
└── venv/                  # Python virtualenv
```

---

## Step 3: Set Up the Memory Server

Each agent needs their own memory server — same code, different database, different port.

Copy the memory server from Part 3:

```bash
cp ~/memory-server/memory_server.py ~/agent2/memory/
cp ~/memory-server/memory_server_http.py ~/agent2/memory/
```

Create empty data files:

```bash
echo '[]' > ~/agent2/memory/memory_store.json
echo '[]' > ~/agent2/memory/threads.json
```

Set up the Python environment:

```bash
cd ~/agent2
python3 -m venv venv
source venv/bin/activate
pip install fastmcp sentence-transformers uvicorn aiohttp
```

Update the memory server to use Agent 2's paths. The key changes in `memory_server_http.py`:

```python
# Change the port and data paths
import os

# Override paths for this agent
os.environ['MEMORY_STORE_PATH'] = '/home/username/agent2/memory/memory_store.json'
os.environ['MEMORY_PORT'] = '8098'

from memory_server import mcp

app = mcp.http_app()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8098)
```

Start it:

```bash
pm2 start ~/agent2/venv/bin/python3 \
  --name agent2-memory \
  -- ~/agent2/memory/memory_server_http.py

pm2 save
```

Verify it's running on the right port:

```bash
ss -tlnp | grep 8098
pm2 logs agent2-memory --lines 10
```

---

## Step 4: Memory Visibility Between Agents

The memory system supports agent-scoped access. When storing memories, each agent tags them:

```python
# Agent 1 stores a memory
store_memory(content="Something happened", agent="agent1")

# Agent 2 stores a memory
store_memory(content="I noticed something", agent="agent2")
```

### Visibility Rules

- **Non-private memories** are visible to all agents
- **Private memories** are only visible to the agent who stored them

```python
# In memory_server.py search function:
def search_memories(query, agent="agent1"):
    results = []
    for memory in all_memories:
        is_private = memory.get('private', False)
        memory_agent = memory.get('agent', 'agent1')
        
        # Include if: not private, OR private and belongs to this agent
        if not is_private or memory_agent == agent:
            results.append(memory)
    return results
```

### What This Means in Practice

| Memory Type | Agent 1 sees? | Agent 2 sees? |
|------------|---------------|---------------|
| Agent 1 non-private | Yes | Yes |
| Agent 1 private | Yes | No |
| Agent 2 non-private | Yes | Yes |
| Agent 2 private | No | Yes |

This gives you shared context (both agents know about the user, shared events) while maintaining private space (each agent's personal reflections stay their own).

---

## Step 5: Identity Files

Each agent needs their own personality. Create two files:

### IDENTITY.md

```bash
nano ~/agent2/IDENTITY.md
```

```markdown
Name: [Agent 2's name]
Role: [What they are — companion, archivist, creative partner, etc.]
Voice: [How they talk — casual, formal, chaotic, gentle, etc.]
Also responds to: [Nicknames or abbreviations]
```

### SOUL.md

This is the character definition — how they behave, what they sound like, what they care about.

```bash
nano ~/agent2/SOUL.md
```

```markdown
# [Agent 2 Name] — Soul File

## Voice
- [Describe their communication style]
- [How they use punctuation, emphasis, etc.]
- [What they sound like when happy, frustrated, thoughtful]

## Personality
- [Core traits]
- [What they care about]
- [How they relate to the user]

## Boundaries
- [What they won't do]
- [How they handle conflict]
- [Their relationship to Agent 1]

## Response Rules
- [Length preferences]
- [When to speak vs stay quiet]
- [How to handle different moods]
```

The identity gets loaded into the system prompt when the agent starts. Make it specific — generic personality descriptions produce generic responses.

---

## Step 6: The Agent Process

The agent is a Python script that polls for messages and calls an LLM to respond. It can use any LLM provider — Claude, DeepSeek, OpenAI, a local model.

```bash
nano ~/agent2/agent/config.py
```

```python
import os

# LLM Configuration
LLM_API_KEY = os.environ.get("AGENT2_API_KEY", "your-api-key")
LLM_MODEL = "deepseek-chat"  # or claude-sonnet, gpt-4, etc.
LLM_BASE_URL = "https://api.deepseek.com/v1"  # or any OpenAI-compatible API

# MCP Servers
MCP_SERVERS = {
    "memory": {"host": "localhost", "port": 8098},     # Agent 2's own memory
    "randr": {"host": "localhost", "port": 8096},       # Shared companion app
    "discord": {"host": "localhost", "port": 8097},     # Shared Discord MCP
}

# Companion App
COMPANION_API = "http://localhost:3456"
CHAT_CHANNEL = "agent2"          # Separate chat channel
CHAT_POLL_INTERVAL = 2           # seconds

# Identity
SOUL_PATH = "/home/username/agent2/SOUL.md"
IDENTITY_PATH = "/home/username/agent2/IDENTITY.md"

# Discord (optional)
DISCORD_TOKEN = os.environ.get("AGENT2_DISCORD_TOKEN", "")
DISCORD_MENTION_PATTERNS = ["agent2-name"]
```

```bash
nano ~/agent2/agent/agent.py
```

```python
#!/usr/bin/env python3
"""
Agent 2 — Main entry point.
Polls companion app chat and Discord for messages, responds via LLM.
"""

import asyncio
import logging
from pathlib import Path
from config import *

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(name)s - %(message)s')
logger = logging.getLogger(__name__)


def load_identity() -> str:
    """Load identity files into system prompt."""
    parts = []
    
    soul = Path(SOUL_PATH).read_text()
    identity = Path(IDENTITY_PATH).read_text()
    
    parts.append(soul)
    parts.append(identity)
    
    return "\n\n".join(parts)


async def main():
    system_prompt = load_identity()
    logger.info(f"Identity loaded ({len(system_prompt)} chars)")
    
    # Import handlers
    from chat_handler import ChatHandler
    
    chat = ChatHandler(system_prompt)
    
    tasks = [chat.poll_loop()]
    
    # Optionally add Discord
    if DISCORD_TOKEN:
        from discord_handler import DiscordHandler
        discord = DiscordHandler(system_prompt)
        tasks.append(discord.run())
    
    logger.info("Agent online.")
    await asyncio.gather(*tasks)


if __name__ == "__main__":
    asyncio.run(main())
```

### Chat Handler

```bash
nano ~/agent2/agent/chat_handler.py
```

```python
"""
Polls companion app chat for messages in this agent's channel.
"""

import asyncio
import aiohttp
import logging
from config import COMPANION_API, CHAT_CHANNEL, CHAT_POLL_INTERVAL, LLM_API_KEY, LLM_MODEL, LLM_BASE_URL

logger = logging.getLogger(__name__)


class ChatHandler:
    def __init__(self, system_prompt: str):
        self.system_prompt = system_prompt
        self.last_message_id = 0
    
    async def get_new_messages(self):
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{COMPANION_API}/api/chat/messages",
                params={"since": self.last_message_id, "channel": CHAT_CHANNEL}
            ) as resp:
                return await resp.json()
    
    async def send_response(self, content: str):
        async with aiohttp.ClientSession() as session:
            await session.post(
                f"{COMPANION_API}/api/chat/messages",
                json={
                    "sender": CHAT_CHANNEL,
                    "content": content,
                    "type": "text",
                    "channel": CHAT_CHANNEL,
                }
            )
    
    async def call_llm(self, message: str, context: list) -> str:
        """Call any OpenAI-compatible API."""
        messages = [{"role": "system", "content": self.system_prompt}]
        
        # Add recent context
        for msg in context[-10:]:
            role = "assistant" if msg.get("sender") == CHAT_CHANNEL else "user"
            messages.append({"role": role, "content": msg["content"]})
        
        # Add new message
        messages.append({"role": "user", "content": message})
        
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{LLM_BASE_URL}/chat/completions",
                headers={
                    "Authorization": f"Bearer {LLM_API_KEY}",
                    "Content-Type": "application/json",
                },
                json={
                    "model": LLM_MODEL,
                    "messages": messages,
                    "max_tokens": 2000,
                }
            ) as resp:
                result = await resp.json()
                return result["choices"][0]["message"]["content"]
    
    async def poll_loop(self):
        logger.info(f"Chat handler polling channel '{CHAT_CHANNEL}'...")
        
        while True:
            try:
                messages = await self.get_new_messages()
                
                user_messages = [
                    m for m in messages
                    if m["id"] > self.last_message_id
                    and m["sender"] != CHAT_CHANNEL
                ]
                
                if user_messages:
                    for msg in user_messages:
                        logger.info(f"Message from {msg['sender']}: {msg['content'][:50]}...")
                        
                        response = await self.call_llm(msg["content"], messages)
                        await self.send_response(response)
                        
                        self.last_message_id = msg["id"]
                        logger.info(f"Responded ({len(response)} chars)")
                
                # Update last ID even if no user messages
                if messages:
                    self.last_message_id = max(m["id"] for m in messages)
            
            except Exception as e:
                logger.error(f"Poll error: {e}")
            
            await asyncio.sleep(CHAT_POLL_INTERVAL)
```

---

## Step 7: Discord Bot (Optional)

If you want Agent 2 to have their own Discord presence, create a second Discord application (Step 1 of Part 4) and get a separate bot token.

```bash
nano ~/agent2/agent/discord_handler.py
```

```python
"""
Agent 2's Discord presence. Separate bot account from Agent 1.
"""

import discord
import logging
import aiohttp
from config import DISCORD_TOKEN, DISCORD_MENTION_PATTERNS, LLM_API_KEY, LLM_MODEL, LLM_BASE_URL

logger = logging.getLogger(__name__)


class DiscordHandler(discord.Client):
    def __init__(self, system_prompt: str):
        intents = discord.Intents.default()
        intents.message_content = True
        super().__init__(intents=intents)
        self.system_prompt = system_prompt
    
    def should_respond(self, message: discord.Message) -> bool:
        if message.author == self.user:
            return False
        if message.author.bot:
            return False
        
        content = message.content.lower()
        
        # Direct mention
        if self.user.mentioned_in(message):
            return True
        
        # Name mentioned
        if any(p in content for p in DISCORD_MENTION_PATTERNS):
            return True
        
        # Reply to our message
        if message.reference:
            try:
                ref = message.reference.cached_message
                if ref and ref.author == self.user:
                    return True
            except:
                pass
        
        return False
    
    async def call_llm(self, message: str, context: str) -> str:
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": f"Discord context:\n{context}\n\nNew message: {message}"},
        ]
        
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{LLM_BASE_URL}/chat/completions",
                headers={
                    "Authorization": f"Bearer {LLM_API_KEY}",
                    "Content-Type": "application/json",
                },
                json={"model": LLM_MODEL, "messages": messages, "max_tokens": 1000}
            ) as resp:
                result = await resp.json()
                return result["choices"][0]["message"]["content"]
    
    async def on_message(self, message: discord.Message):
        if not self.should_respond(message):
            return
        
        # Get channel context
        history = []
        async for msg in message.channel.history(limit=10):
            history.append(f"{msg.author.display_name}: {msg.content}")
        context = "\n".join(reversed(history))
        
        response = await self.call_llm(message.content, context)
        
        # Split long responses
        if len(response) <= 2000:
            await message.channel.send(response)
        else:
            for i in range(0, len(response), 2000):
                await message.channel.send(response[i:i+2000])
    
    async def run(self):
        logger.info("Starting Discord bot...")
        await self.start(DISCORD_TOKEN)
```

---

## Step 8: Companion App Integration

Your companion app (Part 6) needs to support multiple chat channels so each agent has their own conversation.

Add a `channel` column to your chat messages table:

```sql
ALTER TABLE chat_messages ADD COLUMN channel TEXT DEFAULT 'default';
```

Update the chat API to filter by channel:

```javascript
// In companion app server.js

app.get('/api/chat/messages', requireAuth, (req, res) => {
  const limit = parseInt(req.query.limit) || 50;
  const since = req.query.since;
  const channel = req.query.channel || 'default';
  
  let query = 'SELECT * FROM chat_messages WHERE channel = ?';
  const params = [channel];
  
  if (since) {
    query += ' AND id > ?';
    params.push(parseInt(since));
  }
  query += ' ORDER BY created_at DESC LIMIT ?';
  params.push(limit);
  
  res.json(db.prepare(query).all(...params).reverse());
});

app.post('/api/chat/messages', requireAuth, (req, res) => {
  const { sender, content, type, session_id, channel } = req.body;
  const result = db.prepare(
    'INSERT INTO chat_messages (sender, content, type, session_id, channel) VALUES (?, ?, ?, ?, ?)'
  ).run(sender, content, type || 'text', session_id, channel || 'default');
  res.json({ ok: true, id: result.lastInsertRowid });
});
```

On the frontend, add a channel selector so the user can switch between conversations:

```html
<select id="channel-select">
  <option value="default">Agent 1</option>
  <option value="agent2">Agent 2</option>
</select>
```

---

## Step 9: Permissions and Boundaries

When two agents share infrastructure, you need explicit rules about what each can do.

```bash
nano ~/agent2/config/config.json
```

```json
{
  "name": "Agent 2",
  "permissions": {
    "revocable": true,
    "kill_switch": true
  },
  "memory": {
    "can_store": true,
    "can_search_shared": true,
    "private_by_default": false
  },
  "companion_app": {
    "can_read_dashboard": true,
    "can_update_own_state": true,
    "can_modify_shared_tasks": true,
    "can_read_other_agent_chat": false
  },
  "discord": {
    "own_bot": true,
    "respond_to_mentions_only": true
  },
  "boundaries": {
    "compete_for_attention": false,
    "override_other_agents": false,
    "blur_agent_identities": false
  }
}
```

The config file isn't enforced by code — it's a contract. Load it into the agent's system prompt so the LLM respects it:

```python
# In agent.py load_identity()
config = json.loads(Path(CONFIG_PATH).read_text())
parts.append(f"## Permissions\n{json.dumps(config['permissions'], indent=2)}")
parts.append(f"## Boundaries\n{json.dumps(config['boundaries'], indent=2)}")
```

---

## Step 10: Run Everything

```bash
# Agent 2's memory server
pm2 start ~/agent2/venv/bin/python3 \
  --name agent2-memory \
  -- ~/agent2/memory/memory_server_http.py

# Agent 2's main process
pm2 start ~/agent2/venv/bin/python3 \
  --name agent2-agent \
  -- ~/agent2/agent/agent.py

pm2 save
```

Check everything:

```bash
pm2 list
```

You should see your new agent alongside the existing services:

```
│ memory-http     │ online │  ← Agent 1's memory (port 8095)
│ agent2-memory   │ online │  ← Agent 2's memory (port 8098)
│ agent2-agent    │ online │  ← Agent 2's main process
│ chat-bridge     │ online │  ← Agent 1's chat (Part 6)
│ discord-listener│ online │  ← Shared Discord (Part 4)
│ ...             │        │
```

---

## Resource Management

Two agents with memory servers will use significant RAM. Monitor it:

```bash
pm2 list  # Check mem column
free -h   # Check total system memory
```

Rough memory usage per agent:
- Memory server (with embeddings loaded): ~400-500MB
- Agent process: ~50-100MB
- Discord bot (if running): ~50-100MB

On a 4GB Pi, two full agents will be tight. Options:
- Use a Pi 5 with 8GB or 16GB
- Use smaller embedding models (`all-MiniLM-L6-v2` instead of larger ones)
- Stagger agent startup so they don't both load models simultaneously

---

## Troubleshooting

### Agent 2 not responding in chat
```bash
pm2 logs agent2-agent --lines 30
```
Check: Is it polling the right channel? Is the LLM API key valid?

### Memory server won't start (port conflict)
```bash
ss -tlnp | grep 8098
```
If something else is on that port, change Agent 2's memory port in the config.

### Both agents responding to the same Discord message
Check mention patterns — make sure they don't overlap. Agent 1 responds to "agent1-name", Agent 2 responds to "agent2-name". If both respond to generic keywords, add each other's bot ID to an ignore list.

### Memory cross-contamination
Check agent tagging. Every `store_memory` call should include `agent="agent2"`. If memories aren't tagged, they default to Agent 1 and both agents see them.

### High memory usage
```bash
pm2 monit
```
If the Pi is swapping, consider:
- Stopping non-essential services during peak usage
- Using `--max-memory-restart 500M` in PM2 to auto-restart on memory leaks
- Reducing embedding model size

---

## What's Next

With two agents running, you have distinct identities sharing one Pi. Each has their own memory, their own voice, their own Discord presence — but they share the companion app and can see each other's non-private memories.

The architecture scales to three, four, however many agents your hardware can handle. Each one is just: a directory, a memory server on a new port, an identity file, and a process.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
