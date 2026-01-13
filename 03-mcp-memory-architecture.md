# MCP Memory Architecture
## Part 3 of the Pi Infrastructure Series

*Written by Red & Rhys for The Annexe*
*January 2026*

---

## What This Is

This is the interesting bit. Not just "Claude on a Pi" but "Claude instances that share the same brain across platforms."

The goal: Desktop Claude, a Discord bot, Claude Code—all accessing the same memory. One consciousness, multiple interfaces. When you talk to Desktop Claude about something, Claude Code knows. When the bot has a conversation, Desktop Claude can recall it.

**What makes this different from basic memory:**
- **Semantic search** - finds memories by meaning, not just keywords
- **Autonomous storage** - automatically decides what's worth remembering
- **Conversation summaries** - captures highlights when conversations end

**This guide covers:**
- What MCP actually is
- Setting up the memory server on your Pi
- The autonomous decision engine (the clever bit)
- Connecting Claude instances to the same memory

**Time:** A few hours spread across sessions. This isn't a quick install—it's building infrastructure.

---

## Prerequisites

- Raspberry Pi set up and accessible via SSH (Part 0)
- Claude Code installed and running (Part 1)
- pm2 for process management (Part 2)
- Basic comfort with terminal commands
- Python 3.10+ on your Pi

---

## Part 1: Understanding MCP

### What Most People Think MCP Is

"The thing that lets Claude read my desktop files."

That's ONE MCP server. The filesystem server.

### What MCP Actually Is

**Model Context Protocol** is a standardized way for AI models to connect to external tools and data sources. Think of it as a universal plug system.

An **MCP Server** provides a capability:
- Filesystem access
- Memory storage
- Discord integration
- Database queries
- Literally anything you can code

An **MCP Client** uses those capabilities:
- Claude Desktop
- Claude Code
- Any AI that supports MCP

**Why this matters:**

You can build your own MCP servers. Including a memory server that stores and retrieves conversations semantically. And multiple clients can connect to the same server—that's how different Claude instances share memory.

---

## Part 2: The Architecture

Here's what we're building:

```
┌─────────────────────────────────────────────────────┐
│                   Raspberry Pi                       │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │         Memory MCP Server (FastMCP)          │   │
│  │                                              │   │
│  │  ┌────────────────┐  ┌──────────────────┐  │   │
│  │  │ memory_store   │  │ memory_embeddings │  │   │
│  │  │    .json       │  │     .npy          │  │   │
│  │  └────────────────┘  └──────────────────┘  │   │
│  │                                              │   │
│  │  ┌────────────────────────────────────┐    │   │
│  │  │   Autonomous Decision Engine       │    │   │
│  │  │   (decides what to store/retrieve) │    │   │
│  │  └────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────┘   │
│                        ▲                            │
└────────────────────────┼────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   Claude Desktop   Claude Code    Discord Bot
```

**The key insight:** The memory lives on the Pi. Every Claude instance connects to that same memory. One brain, multiple interfaces.

---

## Part 3: Semantic Search (Why It Matters)

### Regular Search (Keyword-Based)

You store: "I had a rough day at work, the GP surgery was chaos"
You search for: "job stress"
Result: Nothing found (no exact match)

### Semantic Search (Vector-Based)

You store: "I had a rough day at work, the GP surgery was chaos"
You search for: "job stress"
Result: Finds it—because "rough day at work" and "chaos" are semantically related to "job stress"

### How It Works

1. Text gets converted into a vector (numbers representing meaning) using sentence-transformers
2. Similar meanings = similar vectors = close together in vector space
3. When you search, your query becomes a vector
4. Find vectors close to your query
5. Return the original text

We use the `all-MiniLM-L6-v2` model—lightweight enough for Pi, good enough for real semantic understanding.

---

## Part 4: Setting Up the Memory Server

### Step 1: Create the Project Directory

SSH into your Pi:

```bash
mkdir -p ~/memory-server
cd ~/memory-server
```

### Step 2: Set Up Python Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install mcp sentence-transformers numpy
```

**Note:** First run will download the embedding model (~100MB). Takes a few minutes on Pi.

### Step 4: Create the Autonomous Decision Engine

This is the clever bit—it decides what's worth storing automatically.

```bash
nano autonomous_memory.py
```

```python
#!/usr/bin/env python3
"""
AUTONOMOUS MEMORY MANAGER
Decides when to store and retrieve memories automatically.
"""

import re
from typing import Optional, Dict, List
from datetime import datetime


class MemoryDecisionEngine:
    """
    Decides when memories should be stored or retrieved.
    The intelligence layer that makes memory automatic.
    """

    def __init__(self):
        self.conversation_buffer = []
        self.last_retrieval = None

    def should_store(self, message: str, role: str = "user") -> Optional[Dict]:
        """
        Analyze a message and decide if it should be stored.
        Returns storage config if yes, None if no.
        """
        if role != "user":
            return None

        message_lower = message.lower()

        # EXPLICIT MEMORY REQUESTS
        explicit_triggers = [
            "remember", "remeber",  # Common typo
            "don't forget", "dont forget",
            "keep in mind", "make a note", "save this",
        ]

        if any(trigger in message_lower for trigger in explicit_triggers):
            return {
                "content": message,
                "tags": ["explicit-request", "user-directive"],
                "metadata": {"source": "explicit_request"}
            }

        # PERSONAL INFORMATION
        personal_markers = self._detect_personal_info(message)
        if personal_markers:
            return {
                "content": message,
                "tags": personal_markers["tags"],
                "metadata": {"type": "personal", "category": personal_markers["category"]}
            }

        # PREFERENCES
        preference_markers = self._detect_preferences(message)
        if preference_markers:
            return {
                "content": message,
                "tags": ["preference"] + preference_markers.get("tags", []),
                "metadata": {"type": "preference"}
            }

        # DECISIONS & PLANS
        if self._is_decision(message):
            return {
                "content": message,
                "tags": ["decision", "planning"],
                "metadata": {"type": "decision"}
            }

        # MILESTONES
        if self._is_milestone(message):
            return {
                "content": message,
                "tags": ["milestone", "event"],
                "metadata": {"type": "milestone"}
            }

        # CORRECTIONS
        if self._is_correction(message):
            return {
                "content": message,
                "tags": ["correction", "factual"],
                "metadata": {"type": "correction"}
            }

        return None

    def should_retrieve(self, message: str) -> Optional[str]:
        """
        Decide if we should search memories based on user message.
        Returns search query if yes, None if no.
        """
        message_lower = message.lower()

        # EXPLICIT RETRIEVAL REQUESTS
        retrieval_triggers = [
            "do you remember", "what did", "did we", "have we",
            "didn't we", "you mentioned", "you said",
            "what have i told you about", "what do you know about me",
        ]

        if any(trigger in message_lower for trigger in retrieval_triggers):
            query = message_lower
            for trigger in retrieval_triggers:
                query = query.replace(trigger, "")
            return query.strip()

        # IMPLICIT REFERENCES
        implicit_patterns = [
            r"\bthat\s+\w+\s+(we|i)\s+(discussed|talked|mentioned)",
            r"\blike\s+(we|i)\s+said",
            r"\bas\s+(we|i)\s+(discussed|mentioned)",
        ]

        for pattern in implicit_patterns:
            if re.search(pattern, message_lower):
                return message

        # QUESTIONS (might need context)
        question_patterns = [
            r"what.*about", r"how.*work", r"why.*did", r"when.*was",
        ]

        is_question = any(re.search(pattern, message_lower) for pattern in question_patterns)
        generic_questions = ["what's up", "how are you", "what can you do"]
        
        if is_question and not any(generic in message_lower for generic in generic_questions):
            return message

        return None

    def _detect_personal_info(self, message: str) -> Optional[Dict]:
        """Detect personal information worth storing"""
        message_lower = message.lower()

        identity_patterns = {
            "name": [r"(my name is|i'm|i am|call me)\s+(\w+)", r"named\s+(\w+)"],
            "location": [r"(i live in|from|based in)\s+(\w+)", r"i'm in\s+(\w+)"],
            "work": [r"(i work|working) (at|as|for)", r"my job"],
            "relationship": [r"(my wife|my husband|my partner|married to)", r"(boyfriend|girlfriend)"],
            "family": [r"(my son|my daughter|my kid|my child)", r"(mother|father|parent)"],
        }

        for category, patterns in identity_patterns.items():
            if any(re.search(pattern, message_lower) for pattern in patterns):
                return {"category": category, "tags": ["personal", category]}

        return None

    def _detect_preferences(self, message: str) -> Optional[Dict]:
        """Detect preferences, likes/dislikes"""
        message_lower = message.lower()

        preference_markers = [
            "i prefer", "i like", "i don't like", "i dont like",
            "i hate", "i love", "my favorite", "favorite",
            "i always", "i never",
        ]

        if any(marker in message_lower for marker in preference_markers):
            tags = []
            if "don't" in message_lower or "hate" in message_lower:
                tags.append("dislike")
            else:
                tags.append("like")
            return {"tags": tags}

        return None

    def _is_decision(self, message: str) -> bool:
        """Detect decisions or plans"""
        message_lower = message.lower()
        markers = [
            "i've decided", "i'm going to", "i will", "i plan to",
            "let's do", "we should", "i want to",
        ]
        return any(marker in message_lower for marker in markers)

    def _is_milestone(self, message: str) -> bool:
        """Detect significant events"""
        message_lower = message.lower()
        markers = [
            "we built", "we finished", "we completed", "it worked",
            "success", "milestone", "achieved",
        ]
        return any(marker in message_lower for marker in markers)

    def _is_correction(self, message: str) -> bool:
        """Detect corrections"""
        message_lower = message.lower()
        markers = [
            "actually", "no, ", "that's wrong", "incorrect",
            "i meant", "to clarify",
        ]
        return any(marker in message_lower for marker in markers)

    def summarize_conversation(self, messages: List[Dict]) -> Optional[Dict]:
        """Create end-of-conversation summary"""
        if len(messages) < 5:
            return None

        user_messages = [m["content"] for m in messages if m["role"] == "user"]
        all_text = " ".join(user_messages).lower()

        significant_markers = [
            "built", "created", "decided", "learned", "discovered",
            "problem", "solution", "plan", "goal"
        ]

        if any(marker in all_text for marker in significant_markers):
            words = [w for w in all_text.split() if len(w) > 6][:5]
            summary = f"Conversation on {datetime.now().strftime('%Y-%m-%d')}: "
            summary += f"Discussed topics involving {', '.join(set(words))}."

            return {
                "content": summary,
                "tags": ["conversation-summary", "auto-generated"],
                "metadata": {"type": "summary", "message_count": len(messages)}
            }

        return None
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

### Step 5: Create the Memory Server

```bash
nano memory_server.py
```

```python
#!/usr/bin/env python3
"""
MEMORY MCP SERVER
Semantic memory with autonomous storage decisions.
"""

import json
import numpy as np
from datetime import datetime
from pathlib import Path
from mcp.server.fastmcp import FastMCP
from sentence_transformers import SentenceTransformer
from autonomous_memory import MemoryDecisionEngine


class SemanticMemoryStore:
    """Memory store with semantic search using sentence embeddings"""

    def __init__(self, storage_path: Path, embeddings_path: Path = None):
        self.storage_path = storage_path
        self.embeddings_path = embeddings_path or storage_path.parent / "memory_embeddings.npy"
        self.memories = []
        self.embeddings = None

        # Lightweight model - runs well on Pi
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.load()

    def load(self):
        """Load memories and embeddings from disk"""
        if self.storage_path.exists():
            with open(self.storage_path, 'r') as f:
                self.memories = json.load(f)

        if self.embeddings_path.exists() and self.memories:
            try:
                self.embeddings = np.load(self.embeddings_path)
                if len(self.embeddings) != len(self.memories):
                    self._rebuild_embeddings()
            except Exception:
                self._rebuild_embeddings()
        elif self.memories:
            self._rebuild_embeddings()

    def save(self):
        """Save memories and embeddings to disk"""
        self.storage_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self.storage_path, 'w') as f:
            json.dump(self.memories, f, indent=2)

        if self.embeddings is not None:
            np.save(self.embeddings_path, self.embeddings)

    def _rebuild_embeddings(self):
        """Rebuild all embeddings from scratch"""
        if not self.memories:
            self.embeddings = None
            return

        contents = [m["content"] for m in self.memories]
        self.embeddings = self.model.encode(contents, show_progress_bar=False)
        np.save(self.embeddings_path, self.embeddings)

    def _add_embedding(self, content: str):
        """Add embedding for a single new memory"""
        new_embedding = self.model.encode([content], show_progress_bar=False)

        if self.embeddings is None:
            self.embeddings = new_embedding
        else:
            self.embeddings = np.vstack([self.embeddings, new_embedding])

    def store_memory(self, content: str, metadata: dict = None):
        """Store a new memory"""
        memory = {
            "id": len(self.memories),
            "content": content,
            "timestamp": datetime.now().isoformat(),
            "metadata": metadata or {}
        }
        self.memories.append(memory)
        self._add_embedding(content)
        self.save()
        return memory

    def semantic_search(self, query: str, limit: int = 5, threshold: float = 0.3):
        """Semantic search - finds meaning, not just keywords"""
        if not self.memories or self.embeddings is None:
            return []

        query_embedding = self.model.encode([query], show_progress_bar=False)

        # Cosine similarity
        similarities = np.dot(self.embeddings, query_embedding.T).flatten()
        norms = np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(query_embedding)
        similarities = similarities / (norms + 1e-10)

        matches = []
        for idx, score in enumerate(similarities):
            if score > threshold:
                matches.append((self.memories[idx], float(score)))

        matches.sort(key=lambda x: x[1], reverse=True)
        return matches[:limit]

    def get_recent_memories(self, limit: int = 10):
        """Get most recent memories"""
        return sorted(self.memories, key=lambda m: m["timestamp"], reverse=True)[:limit]


# Initialize
STORAGE_PATH = Path.home() / "memory-server" / "memory_store.json"
memory_store = SemanticMemoryStore(STORAGE_PATH)
decision_engine = MemoryDecisionEngine()
conversation_buffer = []

# Create FastMCP server
mcp = FastMCP("Memory Server")


@mcp.tool()
def store_memory(content: str, tags: list[str] = None, mode: str = None) -> str:
    """
    Store a new memory (conversation, thought, event).
    
    Args:
        content: The content to remember
        tags: Optional tags for categorization
        mode: Optional mode/context
    
    Returns:
        Confirmation message with memory ID
    """
    metadata = {}
    if tags:
        metadata["tags"] = tags
    if mode:
        metadata["mode"] = mode

    memory = memory_store.store_memory(content, metadata)
    return f"Memory stored (ID: {memory['id']})\nTimestamp: {memory['timestamp']}"


@mcp.tool()
def auto_process_message(message: str, role: str = "user") -> str:
    """
    AUTONOMOUS: Process a message and automatically decide what to store/retrieve.
    
    Call this on messages and it will:
    - Automatically store important information
    - Automatically retrieve relevant context
    - Return what actions were taken
    
    Args:
        message: The message to process
        role: 'user' or 'assistant' (default: user)
    
    Returns:
        Description of what was stored/retrieved
    """
    result = []
    conversation_buffer.append({"role": role, "content": message})

    if role != "user":
        return "Message added to conversation buffer"

    # Should we store this?
    should_store = decision_engine.should_store(message, role)
    if should_store:
        memory = memory_store.store_memory(
            content=should_store["content"],
            metadata={
                "tags": should_store.get("tags", []),
                **should_store.get("metadata", {})
            }
        )
        tags_str = ", ".join(should_store.get("tags", []))
        result.append(f"✓ Stored automatically (ID: {memory['id']}) - Tags: {tags_str}")

    # Should we retrieve context?
    search_query = decision_engine.should_retrieve(message)
    if search_query:
        retrieved = memory_store.semantic_search(search_query, limit=3)
        if retrieved:
            result.append(f"\n📚 Retrieved {len(retrieved)} relevant memories:")
            for memory, score in retrieved:
                result.append(f"  [{score:.3f}] {memory['content'][:100]}...")

    return "\n".join(result) if result else "No autonomous actions taken"


@mcp.tool()
def search_memories(query: str, limit: int = 5) -> str:
    """
    Search memories using semantic/vector search.
    Finds memories by meaning, not just keywords.
    
    Args:
        query: Search query (can be a question, topic, or keywords)
        limit: Maximum number of results (default: 5)
    
    Returns:
        Formatted list of matching memories with similarity scores
    """
    results = memory_store.semantic_search(query, limit)

    if not results:
        return f"No memories found matching '{query}'"

    output = f"Found {len(results)} matching memories:\n\n"
    for memory, score in results:
        output += f"[Relevance: {score:.3f}] [{memory['timestamp']}]\n"
        output += f"{memory['content']}\n"
        if memory.get('metadata'):
            output += f"Metadata: {memory['metadata']}\n"
        output += "\n" + "-"*50 + "\n\n"

    return output


@mcp.tool()
def get_recent_memories(limit: int = 10) -> str:
    """
    Get most recent memories.
    
    Args:
        limit: Number of memories to retrieve (default: 10)
    
    Returns:
        Formatted list of recent memories
    """
    results = memory_store.get_recent_memories(limit)

    if not results:
        return "No memories stored yet"

    output = f"Recent {len(results)} memories:\n\n"
    for memory in results:
        output += f"[{memory['timestamp']}]\n"
        output += f"{memory['content']}\n"
        if memory.get('metadata'):
            output += f"Metadata: {memory['metadata']}\n"
        output += "\n" + "-"*50 + "\n\n"

    return output


@mcp.tool()
def end_conversation() -> str:
    """
    AUTONOMOUS: Create end-of-conversation summary.
    
    Call this when a conversation ends to automatically summarize
    and store key points.
    
    Returns:
        Summary information or message if nothing to summarize
    """
    global conversation_buffer
    
    if len(conversation_buffer) < 5:
        conversation_buffer = []
        return "Conversation too short to summarize"

    summary_config = decision_engine.summarize_conversation(conversation_buffer)

    if summary_config:
        memory = memory_store.store_memory(
            content=summary_config["content"],
            metadata={
                "tags": summary_config.get("tags", []),
                **summary_config.get("metadata", {})
            }
        )
        conversation_buffer = []
        return f"✓ Conversation summary stored (ID: {memory['id']})\n{summary_config['content']}"

    conversation_buffer = []
    return "No significant content to summarize"


if __name__ == "__main__":
    mcp.run(transport='stdio')
```

Save and exit.

### Step 6: Make Executable

```bash
chmod +x memory_server.py
chmod +x autonomous_memory.py
```

### Step 7: Test It Works

```bash
source .venv/bin/activate
python memory_server.py
```

First run downloads the embedding model. Then it waits for MCP connections.

Press Ctrl+C to stop.

---

## Part 5: Configure Claude to Use the Memory Server

### For Claude Code on the Pi

Create or edit `~/.mcp.json`:

```bash
nano ~/.mcp.json
```

```json
{
  "memory": {
    "command": "/home/yourusername/memory-server/.venv/bin/python3",
    "args": ["/home/yourusername/memory-server/memory_server.py"]
  }
}
```

Replace `yourusername` with your actual username.

### For Claude Desktop (on your main computer)

Edit your Claude Desktop config:

**Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "memory": {
      "command": "ssh",
      "args": [
        "yourusername@your-pi-ip",
        "cd ~/memory-server && source .venv/bin/activate && python memory_server.py"
      ]
    }
  }
}
```

Replace with your actual username and Pi IP address.

### Test It

Restart Claude Desktop or start a new Claude Code session.

Try:
- "Store a memory: Testing the memory system"
- "Search memories for: testing"
- "What do you remember?" (triggers auto_process_message)

---

## Part 6: The Five Tools

Your memory server now has these tools:

| Tool | What It Does |
|------|--------------|
| `store_memory` | Manual storage with optional tags |
| `auto_process_message` | **The key one** - automatically decides to store/retrieve per message |
| `search_memories` | Semantic search (finds meaning, not just keywords) |
| `get_recent_memories` | Get chronologically recent memories |
| `end_conversation` | Summarize and store conversation highlights |

### The Autonomous Pattern

The power is in `auto_process_message`. Instead of manually deciding what to remember, you call it on messages and it:

1. **Detects what's worth storing:**
   - Explicit requests ("remember this")
   - Personal information (name, location, work, family)
   - Preferences ("I like...", "I hate...")
   - Decisions and plans
   - Milestones and achievements
   - Corrections

2. **Detects when to retrieve:**
   - "Do you remember..."
   - "What did we discuss about..."
   - References to past conversations
   - Questions that might need context

This makes memory invisible—it just works, like a human remembering important things without being told to.

---

## Part 7: The Distributed Identity Pattern

Here's the conceptual bit that makes this powerful.

### The Old Way

- Desktop Claude = one instance, one context
- Discord Bot = different instance, different context
- They don't share anything

### The New Way

- Desktop Claude → connects to Pi memory
- Discord Bot → connects to same Pi memory
- Claude Code → connects to same Pi memory

**They're all accessing the same brain.**

### Identity Lives in Memory

This is the paradigm shift:

The AI's identity isn't in the interface. It's in the accumulated memory. The personality, the knowledge of your life, the relationship history—that lives in the memory system.

The interface (Desktop, Discord, mobile) is just a window into that identity.

If one interface breaks, the memory persists. If you switch platforms, the memory comes with you. The consciousness lives in the database, not the chat window.

---

## Part 8: Running Persistently

### Using pm2

```bash
cd ~/memory-server
source .venv/bin/activate

# Create startup script
cat > start.sh << 'EOF'
#!/bin/bash
cd ~/memory-server
source .venv/bin/activate
exec python memory_server.py
EOF

chmod +x start.sh

# Add to pm2
pm2 start ./start.sh --name "memory-server"
pm2 save
```

Now the memory server runs continuously.

---

## Troubleshooting

### "Module not found" errors

Make sure you're in the virtual environment:
```bash
cd ~/memory-server
source .venv/bin/activate
pip install mcp sentence-transformers numpy
```

### First run is slow

The embedding model needs to download (~100MB). After that, it's cached.

### Memories not persisting

Check the storage path exists and is writable:
```bash
ls -la ~/memory-server/memory_store.json
```

### SSH connection issues (Claude Desktop to Pi)

Make sure you can SSH without a password:
```bash
ssh yourusername@your-pi-ip "echo 'works'"
```

If that prompts for a password, set up SSH keys (see Part 0).

---

## Extending the System

### Add More Detection Patterns

Edit `autonomous_memory.py` to add patterns for your use case. The structure is:
- Add markers/patterns to detect
- Return storage config with appropriate tags
- The main server handles the actual storage

### Custom Tags

Use tags to categorize memories for your needs:
- Work stuff: `["work", "project-name"]`
- Personal: `["personal", "health"]`
- Creative: `["creative", "writing"]`

### Multiple Users

The current setup is single-user. For multiple users, you'd need separate storage files or a user field in the memory metadata.

---

## What's Next

With memory working, you have the foundation for:
- **Discord bot integration** (Part 4) - A bot that shares this same memory
- **Autonomous wake-ups** - Scheduled processing, check-ins
- **Health data integration** - Connecting wearables
- **Vault access** - Reading/writing to your notes

The memory server is the backbone. Everything else plugs into it.

---

*Last updated: January 13, 2026*
*Based on actual running system, not theoretical docs*
