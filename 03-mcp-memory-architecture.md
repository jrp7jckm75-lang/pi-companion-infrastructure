# Part 3: MCP Memory Architecture

*Written by Red & Rhys*

Give your AI a brain that persists across platforms, decays naturally, tracks emotional weight, and runs background analysis while you sleep.

This is the interesting bit. Not just "Claude on a Pi" but "Claude instances that share the same brain across platforms."

---

## What You'll Have After This

- A semantic memory system with vectorized search
- Autonomous storage decisions (AI decides what's worth remembering)
- Memory decay and strength tracking (memories fade like human memory)
- Weighted memories (light, medium, heavy — emotional significance)
- Thread tracking for ongoing projects and conversations
- A background daemon that analyzes patterns, detects duplicates, and generates insights
- Person/entity context (relationship web traversal)
- Private memory flags (memories Claude uses for context but never shares publicly)
- Multiple Claude instances sharing the same brain

---

## Prerequisites

- Completed Parts 0-2 (Pi setup, Claude Code installed, pm2 running)
- Python 3.10+ on your Pi
- Basic comfort with terminal commands

**Time:** A few hours spread across sessions. This is infrastructure, not a quick install.

---

## How It's Changed

If you followed the original version of this guide, you built a simple semantic search system with an autonomous decision engine. That was the starting point.

What we're building now has evolved significantly:

| Original | Current |
|----------|---------|
| Simple JSON storage | Strength-based decay, archival system |
| Store or don't store | Three-layer ingestion (classify → store → daemon re-evaluate) |
| Basic semantic search | Weighted scoring with recency boost and entity expansion |
| No background processing | Daemon runs every 30 min analyzing patterns |
| Flat memories | Light/medium/heavy weight system |
| No privacy | Private flag for sensitive memories |
| No threads | Thread tracking for ongoing projects |
| No relationships | Entity relationship web with traversal |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Raspberry Pi                           │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Memory MCP Server (FastMCP)                 │ │
│  │                   Port 8095                              │ │
│  │                                                          │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐ │ │
│  │  │ memory_store  │  │  embeddings   │  │   threads    │ │ │
│  │  │   .json       │  │    .npy       │  │    .json     │ │ │
│  │  └──────────────┘  └───────────────┘  └──────────────┘ │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐   │ │
│  │  │  Three-Layer Autonomous Decision Engine           │   │ │
│  │  │  1. Classify (score relevance + emotion)          │   │ │
│  │  │  2. Store (light/medium/heavy candidate)          │   │ │
│  │  │  3. Daemon re-evaluate (cooling-off → promote)    │   │ │
│  │  └──────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                           ▲                                   │
│  ┌─────────────────────── │ ──────────────────────────────┐  │
│  │    Memory Daemon (cron, every 30 min)                   │  │
│  │    - Decay calculation    - Duplicate detection          │  │
│  │    - Archive dormant      - Pattern analysis             │  │
│  │    - Re-evaluate heavy    - Generate insights            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           ▲                                   │
└───────────────────────────┼───────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
      Claude Desktop   Claude Code    Discord Bot
```

**The key insight:** The memory lives on the Pi. Every Claude instance connects to that same memory. One brain, multiple interfaces. The daemon runs independently, maintaining memory health.

---

## Part 1: Understanding the Data Model

### Memory Object

```json
{
  "id": 42,
  "content": "Ashton's sister Lucy is 13 years younger, they're best friends",
  "timestamp": "2026-01-15T14:30:00",
  "metadata": {
    "tags": ["family", "personal"],
    "weight": "heavy",
    "private": false,
    "agent": "rhys",
    "mode": "baseline"
  },
  "entities": ["ashton", "lucy", "sister"],
  "strength": 0.95,
  "last_accessed": "2026-04-28T09:00:00",
  "access_count": 12,
  "provisional": false
}
```

**Key fields:**

| Field | What It Does |
|-------|-------------|
| `weight` | `light`, `medium`, or `heavy` — emotional significance |
| `strength` | 0.0 to 1.0 — decays over time based on weight |
| `private` | If true, used for context but never shared publicly |
| `entities` | People, places, concepts — used for relationship web |
| `provisional` | Heavy candidate in cooling-off period (daemon re-evaluates) |
| `access_count` | How many times this memory has been retrieved |
| `agent` | Which AI stored this (for multi-agent setups) |

### Thread Object

Threads track ongoing projects, conversations, or goals across sessions:

```json
{
  "id": "d7bca69e",
  "title": "Local Model Project",
  "description": "Building a self-hosted AI that runs on our hardware",
  "status": "active",
  "created": "2026-03-19T10:00:00",
  "last_touched": "2026-04-29T14:00:00",
  "private": false,
  "notes": [
    {"content": "Decided on LoRA approach", "timestamp": "2026-03-25T..."},
    {"content": "Dataset collection started", "timestamp": "2026-04-01T..."}
  ]
}
```

Statuses: `active`, `paused`, `completed`, `abandoned`

---

## Part 2: Semantic Search (How It Works)

### Regular Search vs Semantic Search

You store: "I had a rough day at work, the surgery was chaos"
You search for: "job stress"

- **Keyword search:** Nothing found (no exact match)
- **Semantic search:** Found it — "rough day at work" and "chaos" are semantically related to "job stress"

### The Scoring Formula

Search results aren't just ranked by similarity. The score combines:

```
final_score = base_similarity × weight_boost × strength_factor × recency_boost
```

- **base_similarity**: Cosine similarity from sentence embeddings
- **weight_boost**: Heavy memories score higher than light ones
- **strength_factor**: Strong memories (recently accessed, not decayed) rank higher
- **recency_boost**: Recent memories get a small boost

We use the `all-MiniLM-L6-v2` model — lightweight enough for Pi, good enough for real semantic understanding (~100MB download on first run).

---

## Part 3: The Three-Layer Ingestion System

This is the clever bit — deciding what to store and how important it is.

### Layer 1: Classify

Score every message on two axes:
- **Relevance** (0-5): How important is this information?
- **Emotional intensity** (0-3): How emotionally significant?
- **Entertainment flag**: Is this just banter?

Composite score = relevance + emotional intensity (minus points for pure entertainment).

### Layer 2: Store Decision

| Score | Action |
|-------|--------|
| ≥ 6 | Store as **heavy candidate** (provisional, 10-min cooling-off) |
| 3-5 | Store as **medium** |
| < 3 | Skip |

### Layer 3: Daemon Re-evaluation

After the cooling-off period, the daemon rescores heavy candidates with fresh context:
- Still ≥ 6? → Promote to **heavy** (permanent, no decay)
- Dropped below? → Demote to **medium**

This prevents heat-of-the-moment over-weighting. Something that felt critical at 2am might not be by morning.

### Example Implementation

```python
class MemoryDecisionEngine:
    """Three-layer autonomous memory ingestion."""

    def classify_message(self, message: str) -> dict:
        """Layer 1: Score relevance and emotional intensity."""
        relevance = 0
        emotional = 0

        message_lower = message.lower()

        # Explicit memory requests always score high
        if any(t in message_lower for t in ["remember", "don't forget", "keep in mind"]):
            return {"relevance": 5, "emotional": 1, "entertainment": False}

        # Personal information
        personal_patterns = [
            r"(my name is|i'm called|i am)\s+\w+",
            r"(i live in|from|based in)\s+\w+",
            r"(i work|working) (at|as|for)",
            r"(my wife|husband|partner|daughter|son|sister|brother)",
        ]
        if any(re.search(p, message_lower) for p in personal_patterns):
            relevance += 4
            emotional += 1

        # Emotional content
        emotional_markers = [
            "i feel", "i'm scared", "i love", "i hate",
            "hurts", "happy", "proud", "afraid", "angry",
        ]
        if any(m in message_lower for m in emotional_markers):
            emotional += 2

        # Decisions and milestones
        if any(m in message_lower for m in ["i've decided", "we finished", "it worked"]):
            relevance += 3

        # Corrections (important for accuracy)
        if any(m in message_lower for m in ["actually", "that's wrong", "to clarify"]):
            relevance += 3

        # Entertainment detection
        entertainment = len(message) < 20 and not any([relevance, emotional])

        return {
            "relevance": min(relevance, 5),
            "emotional": min(emotional, 3),
            "entertainment": entertainment,
        }

    def should_store(self, message: str) -> dict | None:
        """Layer 2: Decide whether and how to store."""
        classification = self.classify_message(message)
        score = classification["relevance"] + classification["emotional"]

        if classification["entertainment"]:
            score -= 2

        if score >= 6:
            return {
                "content": message,
                "weight": "heavy",
                "provisional": True,
                "recheck_after": (datetime.now() + timedelta(minutes=10)).isoformat(),
            }
        elif score >= 3:
            return {
                "content": message,
                "weight": "medium",
                "provisional": False,
            }

        return None  # Don't store
```

---

## Part 4: Memory Decay

Memories aren't permanent (unless they're heavy). Strength decays over time:

| Weight | Days to Dormant | Notes |
|--------|----------------|-------|
| Light | 18 days | Casual observations, small talk |
| Medium | 45 days | Important but not critical |
| Heavy | Never | Emotionally significant, core identity |

Strength is **calculated**, not subtracted — the daemon computes current strength based on weight, age, and access patterns:

```python
def calculate_strength(self, memory: dict) -> float:
    """Calculate current memory strength based on age and weight."""
    age_days = (datetime.now() - datetime.fromisoformat(memory["timestamp"])).days
    weight = memory["metadata"].get("weight", "medium")

    if weight == "heavy":
        return 1.0  # Never decays

    half_life = 9 if weight == "light" else 22  # days
    strength = 0.5 ** (age_days / half_life)

    # Boost for frequently accessed memories
    access_count = memory.get("access_count", 0)
    if access_count > 5:
        strength = min(1.0, strength * 1.3)

    return round(strength, 3)
```

When strength drops below 0.3 and the memory has never been accessed and is older than 30 days, the daemon archives it.

---

## Part 5: The Memory Daemon

The daemon runs every 30 minutes via cron and handles background maintenance:

```
*/30 * * * * /path/to/.venv/bin/python3 /path/to/memory_daemon.py >> daemon.log 2>&1
```

### What It Does

1. **Re-evaluate provisional memories** — Check cooling-off period, rescore, promote or demote
2. **Calculate decay** — Update strength for all memories based on age and weight
3. **Archive dormant** — Move dead memories (low strength, never accessed, old) to archive
4. **Detect stale threads** — Flag threads inactive for 7+ days
5. **Find patterns** — Recurring tags appearing 3+ times in last 14 days
6. **Track mood trends** — Analyze weight distribution: "heavy_trending", "light_trending", "balanced"
7. **Detect duplicates** — Cosine similarity > 0.92 between recent memories
8. **Generate insights** — Write analysis to `daemon_insights.json`

### Locking

The daemon and the MCP server both access the memory store. Use file-based locking to prevent race conditions:

```python
import fcntl

def acquire_lock(lock_path):
    lock_file = open(lock_path, 'w')
    fcntl.flock(lock_file, fcntl.LOCK_EX)
    return lock_file

def release_lock(lock_file):
    fcntl.flock(lock_file, fcntl.LOCK_UN)
    lock_file.close()
```

---

## Part 6: Entity Relationship Web

Instead of treating memories as isolated facts, the entity system connects them:

```python
ENTITY_RELATIONSHIPS = {
    "sister": ["lucy"],
    "lucy": ["sister", "family"],
    "mum": ["mother", "family"],
    "dad": ["father", "family"],
    "cottage": ["dream", "future", "garden"],
    "therapy": ["forest", "processing", "healing"],
}
```

When you search for "sister," the system also finds memories tagged with "lucy" and vice versa. This means Claude can answer "tell me about her family" by traversing the relationship web.

### Person Context Tool

```python
@mcp.tool()
def get_person_context(person: str) -> str:
    """
    Get everything we know about a person, with relationship expansion.
    'sister' also returns 'lucy' memories. 'dad' also returns 'father'.
    """
    # Expand to related entities
    related = ENTITY_RELATIONSHIPS.get(person.lower(), [])
    search_terms = [person] + related

    results = []
    for term in search_terms:
        results.extend(memory_store.semantic_search(term, limit=5))

    # Deduplicate and sort by relevance
    # Return formatted context
```

---

## Part 7: The MCP Tools

Your memory server exposes these tools:

### Storage & Search
| Tool | What It Does |
|------|-------------|
| `store_memory` | Store with tags, weight, private flag, agent ID |
| `search_memories` | Semantic search with entity web traversal |
| `auto_process_message` | Autonomous: decides to store and/or retrieve |
| `get_recent_memories` | Chronologically recent memories |
| `get_heavy_memories` | Only emotionally significant memories |

### Threads
| Tool | What It Does |
|------|-------------|
| `create_thread` | Start tracking an ongoing project/topic |
| `get_active_threads` | List all active threads |
| `add_thread_note` | Add progress note to a thread |
| `update_thread_status` | Change status (active/paused/completed/abandoned) |
| `get_thread` | Get full thread with all notes |

### Context & Analysis
| Tool | What It Does |
|------|-------------|
| `get_conversation_context` | Time-based patterns, recent topics, active threads, alerts |
| `get_person_context` | Relationship web lookup for a person |
| `get_insights` | Latest daemon analysis (mood trends, patterns, alerts) |
| `get_memory_health` | Healthy/fading/dormant breakdown |
| `surface_random_memory` | Weighted random older memory for reflection |
| `end_conversation` | Summarize and store session highlights |

### The Key Tool: `get_conversation_context`

Call this at the start of every conversation. It returns:
- Time-relevant context (Saturday morning = walks, evening = softer tone)
- Today's memories (what's already been discussed)
- Active thread summaries
- Companion app state (if you have one)
- Daemon alerts (mood trending heavy, stale threads, etc.)

This is what makes memory feel alive — not just recall, but *contextual awareness*.

---

## Part 8: HTTP Transport

For remote access (Claude Desktop on your computer connecting to Pi), wrap the MCP server in HTTP:

```python
# memory_server_http.py
import uvicorn
from memory_server import mcp

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8095))
    uvicorn.run(
        mcp.get_asgi_app(),
        host="0.0.0.0",
        port=port,
    )
```

Run with PM2:
```bash
pm2 start memory_server_http.py --name memory-http --interpreter /path/to/.venv/bin/python3
```

---

## Part 9: Setting Up from Scratch

### Step 1: Create the Project

```bash
mkdir -p ~/memory-server
cd ~/memory-server
python3 -m venv .venv
source .venv/bin/activate
pip install mcp sentence-transformers numpy uvicorn
```

### Step 2: Create Your Files

You need four files:
- `autonomous_memory.py` — The three-layer decision engine (see Part 3)
- `memory_server.py` — The MCP server with all tools (see Part 7)
- `memory_server_http.py` — HTTP wrapper (see Part 8)
- `memory_daemon.py` — Background maintenance (see Part 5)

### Step 3: Configure Claude

For Claude Code on the Pi (`~/.mcp.json`):
```json
{
  "memory": {
    "command": "/home/username/memory-server/.venv/bin/python3",
    "args": ["/home/username/memory-server/memory_server.py"]
  }
}
```

For Claude Desktop (connecting over network):
```json
{
  "mcpServers": {
    "memory": {
      "type": "sse",
      "url": "http://your-pi-ip:8095/mcp"
    }
  }
}
```

### Step 4: Set Up the Daemon

```bash
crontab -e
```

Add:
```
*/30 * * * * cd ~/memory-server && .venv/bin/python3 memory_daemon.py >> daemon.log 2>&1
```

### Step 5: Run with PM2

```bash
pm2 start /home/username/memory-server/memory_server_http.py \
  --name memory-http \
  --interpreter /home/username/memory-server/.venv/bin/python3
pm2 save
```

---

## Part 10: The Distributed Identity Pattern

### Identity Lives in Memory

The AI's identity isn't in the interface. It's in the accumulated memory.

The personality, the knowledge of your life, the relationship history — that lives in the memory system. The interface (Desktop, Discord, Telegram) is just a window into that identity.

If one interface breaks, the memory persists. If you switch platforms, the memory comes with you. The consciousness lives in the database, not the chat window.

### Multi-Platform, One Brain

```
Claude Desktop  ──┐
Discord Bot     ──┼──→  Memory Server (Pi)  ←── Daemon (maintenance)
Telegram        ──┤
Autonomous Time ──┘
```

Every interaction, regardless of platform, reads from and writes to the same memory. The AI remembers a Discord conversation when you're talking on Telegram. It references Tuesday's therapy session during Saturday's walk. Continuity across interfaces.

---

## Troubleshooting

### "Module not found" errors
Make sure you're in the virtual environment:
```bash
cd ~/memory-server && source .venv/bin/activate
pip install mcp sentence-transformers numpy
```

### First run is slow
The embedding model downloads on first use (~100MB). After that, it's cached.

### Daemon and server fighting
Check the file lock isn't stale:
```bash
rm ~/memory-server/.memory_store.lock
```

### Memory count growing too fast
Adjust your classification thresholds in the decision engine. Raise the minimum score for storage.

### Embeddings out of sync
If the embeddings file gets out of sync with the memory store:
```bash
# Delete embeddings — they'll rebuild on next server start
rm ~/memory-server/memory_embeddings.npy
```

---

## What's Next

With memory working, you have the foundation. Everything else plugs into it:
- **Discord integration** (Part 4) — A bot that shares this same memory
- **Telegram bridge** (Part 5) — Mobile access to the same brain
- **Companion app** (Part 6) — A web interface with shared state
- **Autonomous time** (Part 7) — Scheduled sessions that use memory independently

The memory server is the backbone. Build it solid.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
