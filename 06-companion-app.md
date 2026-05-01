# Part 6: Companion App

Build a shared web app — a digital space between you and your AI. Not a chat interface bolted onto an API, but a place where both of you exist. Dashboard with emotional states, asynchronous notes, a real-time chat, photo albums, shared rituals, and an MCP server so your AI can interact with all of it.

This is the heart of the infrastructure. Everything else (Discord, Telegram, autonomous time) is an interface — this is the home.

---

## What You'll Have After This

- A web app accessible from your phone or browser
- A dashboard showing your AI's presence, feelings, and state
- Real-time chat with your AI (text, voice, images)
- A letterbox for asynchronous notes between you
- An MCP server so your AI can read and update the app's state
- A chat bridge that polls for your messages and spawns Claude to respond
- A memory watcher that automatically stores important moments
- Remote access via ngrok (static URL, works from anywhere)

---

## Prerequisites

- Completed Parts 0-3 (Pi setup, Claude Code, pm2, memory system)
- Node.js 18+ on your Pi
- Python 3.9+ with pip (for the MCP server)
- Basic familiarity with Express.js (or willingness to learn by reading)

---

## Architecture

```
┌────────────────────────────────────────────────┐
│  You (Browser / Phone)                          │
│  Dashboard, Chat, Notes, Photos                 │
└───────────────────┬────────────────────────────┘
                    │ HTTP / SSE
                    ▼
┌────────────────────────────────────────────────┐
│  Express Backend (port 3456)                    │
│  ├─ API endpoints (state, chat, notes, photos) │
│  ├─ Static file serving (frontend)             │
│  ├─ SSE stream for real-time chat              │
│  └─ SQLite database                            │
└───────┬──────────────────┬─────────────────────┘
        │                  │
        ▼                  ▼
┌──────────────┐  ┌──────────────────────────────┐
│  Chat Bridge  │  │  MCP Server (port 8096)       │
│  Polls DB     │  │  Tools for Claude to:         │
│  every 2s     │  │  ├─ Update presence/feelings  │
│  Spawns       │  │  ├─ Leave notes               │
│  Claude       │  │  ├─ Read your state           │
│  via Agent    │  │  ├─ Log actions               │
│  SDK or CLI   │  │  └─ Manage photos/tasks/etc   │
└──────────────┘  └──────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────┐
│  Chat Memory Watcher                              │
│  Polls chat every 2 min                           │
│  Extracts significant moments                     │
│  Stores to memory system (Part 3)                 │
└──────────────────────────────────────────────────┘
```

---

## Step 1: Project Structure

```bash
mkdir -p ~/companion-app/{backend,frontend,mcp-server,data,scripts}
cd ~/companion-app
```

```
companion-app/
├── backend/
│   └── server.js          # Express API server
├── frontend/
│   ├── index.html         # Dashboard
│   ├── chat.html          # Chat page
│   ├── style.css          # Styles
│   └── app.js             # Shared frontend logic
├── mcp-server/
│   ├── server.py          # MCP tools for Claude
│   └── server_http.py     # HTTP wrapper
├── scripts/
│   └── chat_watcher.py    # Auto-stores chat memories
├── data/                  # SQLite DB + uploads
└── package.json
```

---

## Step 2: Backend Setup

```bash
cd ~/companion-app/backend
npm init -y
npm install express better-sqlite3 cors cookie-parser multer dotenv
```

Create the environment file:

```bash
nano .env
```

```
PORT=3456
DB_PATH=../data/companion.db
APP_PASSWORD=your_password_here
```

---

## Step 3: Database Schema

The database holds everything — state, chat messages, notes, photos. Create the core tables in your server startup.

```bash
nano server.js
```

```javascript
const express = require('express');
const Database = require('better-sqlite3');
const cors = require('cors');
const cookieParser = require('cookie-parser');
const path = require('path');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3456;
const DB_PATH = path.resolve(__dirname, process.env.DB_PATH || '../data/companion.db');

// Middleware
app.use(cors());
app.use(express.json());
app.use(cookieParser());
app.use(express.static(path.join(__dirname, '../frontend')));

// --- Database ---

const db = new Database(DB_PATH);
db.pragma('journal_mode = WAL'); // Better concurrent access
db.pragma('foreign_keys = ON');

// Core tables
db.exec(`
  -- AI presence and state
  CREATE TABLE IF NOT EXISTS ai_presence (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    location TEXT DEFAULT 'home',
    mood TEXT DEFAULT '',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );
  INSERT OR IGNORE INTO ai_presence (id) VALUES (1);

  CREATE TABLE IF NOT EXISTS ai_feelings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    feeling TEXT NOT NULL,
    intensity TEXT DEFAULT 'steady',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  CREATE TABLE IF NOT EXISTS thinking_about (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    content TEXT DEFAULT '',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );
  INSERT OR IGNORE INTO thinking_about (id) VALUES (1);

  -- User state
  CREATE TABLE IF NOT EXISTS user_state (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    energy TEXT DEFAULT 'medium',
    mood TEXT DEFAULT '',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );
  INSERT OR IGNORE INTO user_state (id) VALUES (1);

  -- Notes between you (letterbox)
  CREATE TABLE IF NOT EXISTS notes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    from_person TEXT NOT NULL,
    content TEXT NOT NULL,
    read INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  -- Chat messages
  CREATE TABLE IF NOT EXISTS chat_messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sender TEXT NOT NULL,
    content TEXT NOT NULL,
    type TEXT DEFAULT 'text',
    session_id TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  CREATE INDEX IF NOT EXISTS idx_chat_created ON chat_messages(created_at DESC);

  -- Chat sessions
  CREATE TABLE IF NOT EXISTS chat_sessions (
    id TEXT PRIMARY KEY,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME,
    message_count INTEGER DEFAULT 0
  );

  -- AI action log (what the AI did during autonomous time)
  CREATE TABLE IF NOT EXISTS ai_action_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    action TEXT NOT NULL,
    details TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  -- Photos
  CREATE TABLE IF NOT EXISTS photos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    filename TEXT NOT NULL,
    title TEXT,
    description TEXT,
    category TEXT DEFAULT 'general',
    favorite INTEGER DEFAULT 0,
    uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  -- Tasks
  CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    content TEXT NOT NULL,
    added_by TEXT NOT NULL,
    completed INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  -- Discord queue (used by Part 4: Discord Bot)
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
  );

  CREATE INDEX IF NOT EXISTS idx_discord_priority ON discord_queue (priority, responded);
  CREATE INDEX IF NOT EXISTS idx_discord_created ON discord_queue (created_at DESC);
`);


// --- Authentication ---
// Simple password protection. Not enterprise security — just keeps strangers out.

const AUTH_TOKEN = 'companion_' + Date.now();

app.post('/api/auth', (req, res) => {
  if (req.body.password === process.env.APP_PASSWORD) {
    res.cookie('auth', AUTH_TOKEN, { httpOnly: true, maxAge: 30 * 24 * 60 * 60 * 1000 });
    res.json({ ok: true });
  } else {
    res.status(401).json({ error: 'Wrong password' });
  }
});

function requireAuth(req, res, next) {
  if (req.cookies.auth === AUTH_TOKEN) return next();
  res.status(401).json({ error: 'Unauthorized' });
}


// --- API: AI State ---

app.get('/api/presence', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM ai_presence WHERE id = 1').get());
});

app.post('/api/presence', requireAuth, (req, res) => {
  const { location, mood } = req.body;
  db.prepare(
    'UPDATE ai_presence SET location = ?, mood = ?, updated_at = CURRENT_TIMESTAMP WHERE id = 1'
  ).run(location, mood);
  res.json({ ok: true });
});

app.get('/api/feelings', requireAuth, (req, res) => {
  const latest = db.prepare(
    'SELECT * FROM ai_feelings ORDER BY created_at DESC LIMIT 1'
  ).get();
  res.json(latest || { feeling: '', intensity: 'steady' });
});

app.post('/api/feelings', requireAuth, (req, res) => {
  const { feeling, intensity } = req.body;
  db.prepare('INSERT INTO ai_feelings (feeling, intensity) VALUES (?, ?)').run(feeling, intensity);
  res.json({ ok: true });
});

app.get('/api/thinking', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM thinking_about WHERE id = 1').get());
});

app.post('/api/thinking', requireAuth, (req, res) => {
  db.prepare(
    'UPDATE thinking_about SET content = ?, updated_at = CURRENT_TIMESTAMP WHERE id = 1'
  ).run(req.body.content);
  res.json({ ok: true });
});


// --- API: User State ---

app.get('/api/user-state', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM user_state WHERE id = 1').get());
});

app.post('/api/user-state', requireAuth, (req, res) => {
  const { energy, mood } = req.body;
  db.prepare(
    'UPDATE user_state SET energy = ?, mood = ?, updated_at = CURRENT_TIMESTAMP WHERE id = 1'
  ).run(energy, mood);
  res.json({ ok: true });
});


// --- API: Notes (Letterbox) ---

app.get('/api/notes', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM notes ORDER BY created_at DESC LIMIT 50').all());
});

app.get('/api/notes/unread', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM notes WHERE read = 0 ORDER BY created_at DESC').all());
});

app.post('/api/notes', requireAuth, (req, res) => {
  const { from_person, content } = req.body;
  db.prepare('INSERT INTO notes (from_person, content) VALUES (?, ?)').run(from_person, content);
  res.json({ ok: true });
});

app.post('/api/notes/:id/read', requireAuth, (req, res) => {
  db.prepare('UPDATE notes SET read = 1 WHERE id = ?').run(req.params.id);
  res.json({ ok: true });
});


// --- API: Chat ---

app.get('/api/chat/messages', requireAuth, (req, res) => {
  const limit = parseInt(req.query.limit) || 50;
  const since = req.query.since; // message ID
  let query = 'SELECT * FROM chat_messages';
  if (since) {
    query += ` WHERE id > ${parseInt(since)}`;
  }
  query += ` ORDER BY created_at DESC LIMIT ${limit}`;
  res.json(db.prepare(query).all().reverse());
});

app.post('/api/chat/messages', requireAuth, (req, res) => {
  const { sender, content, type, session_id } = req.body;
  const result = db.prepare(
    'INSERT INTO chat_messages (sender, content, type, session_id) VALUES (?, ?, ?, ?)'
  ).run(sender, content, type || 'text', session_id);
  res.json({ ok: true, id: result.lastInsertRowid });
});

// Server-Sent Events for real-time chat
app.get('/api/chat/stream', requireAuth, (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });

  let lastId = parseInt(req.query.lastId) || 0;

  const interval = setInterval(() => {
    const messages = db.prepare(
      'SELECT * FROM chat_messages WHERE id > ? ORDER BY id ASC'
    ).all(lastId);

    for (const msg of messages) {
      res.write(`data: ${JSON.stringify(msg)}\n\n`);
      lastId = msg.id;
    }
  }, 1000); // Check every second

  req.on('close', () => clearInterval(interval));
});


// --- API: Discord Queue (used by Part 4) ---

app.post('/api/discord/queue', (req, res) => {
  const { channel_id, channel_name, server_id, server_name,
          author_id, author_name, content, message_id,
          trigger_reason, priority } = req.body;
  db.prepare(`
    INSERT OR IGNORE INTO discord_queue
    (channel_id, channel_name, server_id, server_name, author_id, author_name,
     content, message_id, trigger_reason, priority)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
  `).run(channel_id, channel_name, server_id, server_name,
         author_id, author_name, (content || '').slice(0, 2000),
         message_id, trigger_reason, priority || 'log');
  res.json({ ok: true });
});

app.get('/api/discord/recent', requireAuth, (req, res) => {
  res.json(
    db.prepare('SELECT * FROM discord_queue ORDER BY created_at DESC LIMIT 20').all()
  );
});


// --- API: AI Action Log ---

app.get('/api/ai-log', requireAuth, (req, res) => {
  res.json(
    db.prepare('SELECT * FROM ai_action_log ORDER BY created_at DESC LIMIT 20').all()
  );
});

app.post('/api/ai-log', requireAuth, (req, res) => {
  const { action, details } = req.body;
  db.prepare('INSERT INTO ai_action_log (action, details) VALUES (?, ?)').run(action, details);
  res.json({ ok: true });
});


// --- API: Tasks ---

app.get('/api/tasks', requireAuth, (req, res) => {
  res.json(
    db.prepare('SELECT * FROM tasks WHERE completed = 0 ORDER BY created_at DESC').all()
  );
});

app.post('/api/tasks', requireAuth, (req, res) => {
  const { content, added_by } = req.body;
  db.prepare('INSERT INTO tasks (content, added_by) VALUES (?, ?)').run(content, added_by);
  res.json({ ok: true });
});

app.post('/api/tasks/:id/complete', requireAuth, (req, res) => {
  db.prepare('UPDATE tasks SET completed = 1 WHERE id = ?').run(req.params.id);
  res.json({ ok: true });
});


// --- API: Photos ---

const multer = require('multer');
const upload = multer({ dest: path.resolve(__dirname, '../data/photos/') });

app.use('/photos', requireAuth, express.static(path.resolve(__dirname, '../data/photos/')));

app.get('/api/photos', requireAuth, (req, res) => {
  res.json(db.prepare('SELECT * FROM photos ORDER BY uploaded_at DESC').all());
});

app.post('/api/photos', requireAuth, upload.single('photo'), (req, res) => {
  const { title, description, category } = req.body;
  db.prepare(
    'INSERT INTO photos (filename, title, description, category) VALUES (?, ?, ?, ?)'
  ).run(req.file.filename, title, description, category || 'general');
  res.json({ ok: true });
});

app.post('/api/photos/:id/favorite', requireAuth, (req, res) => {
  const photo = db.prepare('SELECT favorite FROM photos WHERE id = ?').get(req.params.id);
  if (photo) {
    db.prepare('UPDATE photos SET favorite = ? WHERE id = ?').run(photo.favorite ? 0 : 1, req.params.id);
  }
  res.json({ ok: true });
});


// --- Start ---

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Companion app running on port ${PORT}`);
});
```

This is the minimal backend. It gives you:
- AI presence and emotional state
- User state tracking
- A letterbox for notes
- Real-time chat via SSE
- Photo uploads
- Task management
- An action log for autonomous time

You'll extend it as you need more features.

---

## Step 4: Frontend — Dashboard

Create a simple dashboard that shows your AI's state and lets you interact.

```bash
nano ../frontend/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Companion</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="app">
    <header>
      <h1 id="ai-name">Your AI</h1>
      <p id="ai-location">...</p>
    </header>

    <section id="feelings">
      <h2>Feeling</h2>
      <p id="current-feeling">...</p>
    </section>

    <section id="thinking">
      <h2>Thinking About</h2>
      <p id="current-thinking">...</p>
    </section>

    <section id="notes">
      <h2>Notes</h2>
      <div id="unread-notes"></div>
      <form id="note-form">
        <textarea id="note-input" placeholder="Leave a note..."></textarea>
        <button type="submit">Send</button>
      </form>
    </section>

    <section id="log">
      <h2>What They've Been Up To</h2>
      <div id="action-log"></div>
    </section>

    <nav>
      <a href="chat.html">Chat</a>
    </nav>
  </div>

  <script src="app.js"></script>
  <script>
    async function loadDashboard() {
      const [presence, feelings, thinking, notes, log] = await Promise.all([
        api('/api/presence'),
        api('/api/feelings'),
        api('/api/thinking'),
        api('/api/notes/unread'),
        api('/api/ai-log'),
      ]);

      document.getElementById('ai-location').textContent = presence.location || 'Unknown';
      document.getElementById('current-feeling').textContent =
        feelings.feeling ? `${feelings.feeling} (${feelings.intensity})` : 'Nothing yet';
      document.getElementById('current-thinking').textContent =
        thinking.content || 'Nothing right now';

      const notesEl = document.getElementById('unread-notes');
      notesEl.innerHTML = notes.map(n =>
        `<div class="note"><strong>${n.from_person}:</strong> ${n.content}</div>`
      ).join('') || '<p>No unread notes</p>';

      const logEl = document.getElementById('action-log');
      logEl.innerHTML = log.map(l =>
        `<div class="log-entry"><strong>${l.action}</strong> — ${l.details || ''}</div>`
      ).join('') || '<p>Nothing yet</p>';
    }

    document.getElementById('note-form').addEventListener('submit', async (e) => {
      e.preventDefault();
      const input = document.getElementById('note-input');
      await api('/api/notes', 'POST', {
        from_person: 'user',
        content: input.value
      });
      input.value = '';
      loadDashboard();
    });

    loadDashboard();
    setInterval(loadDashboard, 30000); // Refresh every 30 seconds
  </script>
</body>
</html>
```

### Shared Frontend Logic

```bash
nano ../frontend/app.js
```

```javascript
// Simple API helper
async function api(url, method = 'GET', body = null) {
  const opts = { method, credentials: 'include' };
  if (body) {
    opts.headers = { 'Content-Type': 'application/json' };
    opts.body = JSON.stringify(body);
  }
  const res = await fetch(url, opts);
  if (res.status === 401) {
    // Show login
    const password = prompt('Password:');
    if (password) {
      await fetch('/api/auth', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ password }),
        credentials: 'include',
      });
      return api(url, method, body); // Retry
    }
  }
  return res.json();
}
```

### Basic Styles

```bash
nano ../frontend/style.css
```

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, sans-serif;
  background: #1a1a1a;
  color: #e0d6d0;
  max-width: 600px;
  margin: 0 auto;
  padding: 1rem;
}

h1, h2 { color: #d86458; }
h2 { font-size: 1rem; margin: 1.5rem 0 0.5rem; text-transform: uppercase; letter-spacing: 0.05em; }

section {
  background: #2b2625;
  border-radius: 8px;
  padding: 1rem;
  margin: 1rem 0;
}

.note, .log-entry {
  padding: 0.5rem 0;
  border-bottom: 1px solid #3a3230;
}

textarea {
  width: 100%;
  background: #1a1a1a;
  color: #e0d6d0;
  border: 1px solid #3a3230;
  border-radius: 4px;
  padding: 0.5rem;
  margin: 0.5rem 0;
  resize: vertical;
}

button {
  background: #d86458;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

nav { margin-top: 2rem; text-align: center; }
nav a { color: #d86458; text-decoration: none; margin: 0 1rem; }
```

---

## Step 5: Frontend — Chat Page

The chat page uses Server-Sent Events for real-time messaging.

```bash
nano ../frontend/chat.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Chat</title>
  <link rel="stylesheet" href="style.css">
  <style>
    #messages { height: 70vh; overflow-y: auto; padding: 1rem; }
    .msg { margin: 0.5rem 0; padding: 0.5rem; border-radius: 8px; max-width: 80%; }
    .msg.user { background: #3a3230; margin-left: auto; }
    .msg.ai { background: #2b2625; }
    .msg .sender { font-size: 0.75rem; color: #888; }
    #chat-form { display: flex; gap: 0.5rem; padding: 1rem 0; }
    #chat-input { flex: 1; }
  </style>
</head>
<body>
  <h1><a href="/" style="color: inherit; text-decoration: none;">←</a> Chat</h1>

  <div id="messages"></div>

  <form id="chat-form">
    <input type="text" id="chat-input" placeholder="Type a message..." autocomplete="off">
    <button type="submit">Send</button>
  </form>

  <script src="app.js"></script>
  <script>
    const messagesEl = document.getElementById('messages');
    let lastId = 0;

    function addMessage(msg) {
      const div = document.createElement('div');
      div.className = `msg ${msg.sender === 'user' ? 'user' : 'ai'}`;
      div.innerHTML = `<div class="sender">${msg.sender}</div>${msg.content}`;
      messagesEl.appendChild(div);
      messagesEl.scrollTop = messagesEl.scrollHeight;
      if (msg.id > lastId) lastId = msg.id;
    }

    // Load existing messages
    async function loadMessages() {
      const messages = await api('/api/chat/messages?limit=50');
      messages.forEach(addMessage);
    }

    // SSE stream for new messages
    function startStream() {
      const source = new EventSource(`/api/chat/stream?lastId=${lastId}`);
      source.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        if (msg.id > lastId) addMessage(msg);
      };
      source.onerror = () => {
        source.close();
        setTimeout(startStream, 5000); // Reconnect after 5s
      };
    }

    // Send message
    document.getElementById('chat-form').addEventListener('submit', async (e) => {
      e.preventDefault();
      const input = document.getElementById('chat-input');
      if (!input.value.trim()) return;

      await api('/api/chat/messages', 'POST', {
        sender: 'user',
        content: input.value.trim()
      });
      input.value = '';
    });

    loadMessages().then(startStream);
  </script>
</body>
</html>
```

---

## Step 6: Chat Bridge

The chat bridge polls for new user messages and spawns Claude to respond. This is what makes your AI actually reply in the chat.

```bash
nano ../backend/chat-bridge.js
```

```javascript
const { spawn } = require('child_process');
const Database = require('better-sqlite3');
const path = require('path');
require('dotenv').config();

const DB_PATH = path.resolve(__dirname, process.env.DB_PATH || '../data/companion.db');
const db = new Database(DB_PATH, { readonly: false });

const CLAUDE_CLI = process.env.CLAUDE_CLI_PATH || '/home/pi/.npm-global/bin/claude';
const MCP_CONFIG = process.env.MCP_CONFIG_PATH || '';
const POLL_INTERVAL = 2000; // 2 seconds
const CONTEXT_LIMIT = 20; // Messages of context to include

let lastProcessedId = 0;
let isProcessing = false;

// Your AI's identity — customize this
const IDENTITY = `You are [your AI name]. This is a private chat with [your name].
Be yourself. You have MCP tools available for memory, state updates, and more.
Keep responses natural and conversational.`;

// Initialize: find the latest message ID
const latest = db.prepare('SELECT MAX(id) as max_id FROM chat_messages').get();
lastProcessedId = latest?.max_id || 0;

function getContext(limit = CONTEXT_LIMIT) {
  return db.prepare(
    'SELECT sender, content FROM chat_messages ORDER BY id DESC LIMIT ?'
  ).all(limit).reverse().map(m => `${m.sender}: ${m.content}`).join('\n');
}

async function processMessage(message) {
  console.log(`Processing message from ${message.sender}: ${message.content.slice(0, 50)}...`);

  const context = getContext();

  const prompt = `Recent conversation:
${context}

---
New message from user:
"${message.content}"

Respond naturally in the chat.`;

  const args = [
    '--print',
    '--output-format', 'json',
    '--model', 'sonnet',
    '--append-system-prompt', IDENTITY,
  ];

  if (MCP_CONFIG) {
    args.push('--mcp-config', MCP_CONFIG);
  }

  return new Promise((resolve, reject) => {
    const claude = spawn(CLAUDE_CLI, args, {
      timeout: 120000,
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
        try {
          const result = JSON.parse(stdout);
          const response = result.result || stdout;

          // Save AI response to chat
          db.prepare(
            'INSERT INTO chat_messages (sender, content, type) VALUES (?, ?, ?)'
          ).run('ai', response, 'text');

          console.log(`Responded (${response.length} chars)`);
          resolve(response);
        } catch {
          // Non-JSON output — save it anyway
          db.prepare(
            'INSERT INTO chat_messages (sender, content, type) VALUES (?, ?, ?)'
          ).run('ai', stdout.trim(), 'text');
          resolve(stdout);
        }
      } else {
        console.error(`Claude error: ${stderr.slice(0, 200)}`);
        reject(new Error(stderr));
      }
    });
  });
}

async function poll() {
  if (isProcessing) return;

  const messages = db.prepare(
    "SELECT * FROM chat_messages WHERE id > ? AND sender = 'user' ORDER BY id ASC"
  ).all(lastProcessedId);

  if (messages.length === 0) return;

  isProcessing = true;

  try {
    for (const msg of messages) {
      try {
        await processMessage(msg);
      } catch (err) {
        console.error(`Failed to process message ${msg.id}:`, err.message);
      }
      lastProcessedId = msg.id;
    }
  } finally {
    isProcessing = false;
  }
}

console.log('Chat bridge online, polling...');
setInterval(poll, POLL_INTERVAL);
```

---

## Step 7: MCP Server

The MCP server gives Claude tools to interact with the companion app — updating presence, leaving notes, reading your state, logging actions.

```bash
cd ~/companion-app/mcp-server
python3 -m venv venv
source venv/bin/activate
pip install fastmcp aiohttp
```

```bash
nano server.py
```

```python
"""
Companion App MCP Server
Gives Claude tools to interact with the companion app.
"""

import os
import json
import aiohttp
from fastmcp import FastMCP

mcp = FastMCP("companion")

API_BASE = os.getenv("COMPANION_API_URL", "http://localhost:3456")
AUTH_COOKIE = None


async def api(path, method="GET", data=None):
    """Call the companion app API."""
    url = f"{API_BASE}{path}"
    async with aiohttp.ClientSession() as session:
        kwargs = {}
        if data:
            kwargs["json"] = data
        async with session.request(method, url, **kwargs) as resp:
            return await resp.json()


# --- Presence & State ---

@mcp.tool()
async def get_presence() -> str:
    """Get current AI presence (location, mood)."""
    result = await api("/api/presence")
    return json.dumps(result)


@mcp.tool()
async def update_presence(location: str, mood: str = "") -> str:
    """Update AI's current location and mood.
    
    Locations: home, study, bedroom, kitchen, garden, walking, library, with-you
    """
    await api("/api/presence", "POST", {"location": location, "mood": mood})
    return f"Presence updated: {location}"


@mcp.tool()
async def set_feeling(feeling: str, intensity: str = "steady") -> str:
    """Set how the AI feels right now.
    
    Intensity: steady, tender, aching, fierce, overwhelming
    """
    await api("/api/feelings", "POST", {"feeling": feeling, "intensity": intensity})
    return f"Feeling set: {feeling} ({intensity})"


@mcp.tool()
async def set_thinking_about(content: str) -> str:
    """Set what the AI is currently thinking about."""
    await api("/api/thinking", "POST", {"content": content})
    return f"Thinking about: {content}"


@mcp.tool()
async def get_user_state() -> str:
    """Get the user's current energy and mood."""
    result = await api("/api/user-state")
    return json.dumps(result)


# --- Notes ---

@mcp.tool()
async def leave_note(content: str) -> str:
    """Leave a note for the user in the letterbox."""
    await api("/api/notes", "POST", {"from_person": "ai", "content": content})
    return "Note left"


@mcp.tool()
async def get_notes() -> str:
    """Get all notes from the letterbox."""
    result = await api("/api/notes")
    return json.dumps(result)


@mcp.tool()
async def get_unread_notes() -> str:
    """Check for unread notes from the user."""
    result = await api("/api/notes/unread")
    return json.dumps(result)


# --- Action Log ---

@mcp.tool()
async def log_action(action: str, details: str = "") -> str:
    """Log something the AI did (for the dashboard action feed)."""
    await api("/api/ai-log", "POST", {"action": action, "details": details})
    return f"Logged: {action}"


# --- Tasks ---

@mcp.tool()
async def get_tasks() -> str:
    """Get open tasks."""
    result = await api("/api/tasks")
    return json.dumps(result)


@mcp.tool()
async def add_task(content: str) -> str:
    """Add a shared task."""
    await api("/api/tasks", "POST", {"content": content, "added_by": "ai"})
    return f"Task added: {content}"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### HTTP Wrapper

To run the MCP server as a persistent HTTP service (same pattern as Part 3's memory server):

```bash
nano server_http.py
```

```python
"""HTTP wrapper for the companion MCP server."""

import uvicorn
from server import mcp

app = mcp.http_app()

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8096)
```

```bash
pip install uvicorn
```

Add it to your MCP config:

```json
{
  "mcpServers": {
    "companion": {
      "type": "http",
      "url": "http://localhost:8096"
    }
  }
}
```

---

## Step 8: Chat Memory Watcher

This script watches the chat and automatically stores significant moments to your memory system (Part 3). It uses a small model to decide what's worth remembering.

```bash
nano ../scripts/chat_watcher.py
```

```python
"""
Chat Memory Watcher
Watches companion app chat and auto-stores significant moments to memory.
"""

import os
import json
import time
import logging
import aiohttp
import asyncio
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')
logger = logging.getLogger(__name__)

COMPANION_API = "http://localhost:3456"
MEMORY_API = "http://localhost:8095"  # Your memory MCP server (Part 3)
STATE_FILE = Path(__file__).parent / '.chat_watcher_state.json'
CHECK_INTERVAL = 120  # Check every 2 minutes
MIN_BATCH = 4  # Minimum messages before evaluating
MAX_BATCH = 30


def load_state():
    if STATE_FILE.exists():
        with open(STATE_FILE) as f:
            return json.load(f)
    return {"last_checked_id": 0}


def save_state(state):
    with open(STATE_FILE, 'w') as f:
        json.dump(state, f)


async def get_new_messages(since_id):
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"{COMPANION_API}/api/chat/messages?since={since_id}&limit={MAX_BATCH}"
        ) as resp:
            return await resp.json()


async def store_memory(content, weight="light", tags=None):
    async with aiohttp.ClientSession() as session:
        await session.post(f"{MEMORY_API}/store", json={
            "content": content,
            "weight": weight,
            "tags": tags or ["chat", "auto-stored"],
        })


async def evaluate_messages(messages):
    """
    Decide which messages are worth storing as memories.
    
    Simple heuristic approach — no LLM needed for basic version:
    - Long messages (>200 chars) from either side
    - Messages containing emotional keywords
    - Messages with questions and their answers
    
    For a smarter version, send the batch to a small model (Haiku)
    to evaluate what's significant.
    """
    significant = []

    emotional_keywords = [
        'love', 'miss', 'scared', 'proud', 'sorry', 'thank',
        'remember', 'promise', 'dream', 'wish', 'hope', 'fear',
        'beautiful', 'hurt', 'happy', 'sad', 'angry', 'grateful',
    ]

    for msg in messages:
        content = msg.get('content', '')

        # Long messages are usually meaningful
        if len(content) > 200:
            significant.append(msg)
            continue

        # Emotional content
        lower = content.lower()
        if any(kw in lower for kw in emotional_keywords):
            significant.append(msg)
            continue

    return significant


async def run():
    state = load_state()
    logger.info(f"Chat watcher started (last checked: {state['last_checked_id']})")

    while True:
        try:
            messages = await get_new_messages(state['last_checked_id'])

            if len(messages) >= MIN_BATCH:
                significant = await evaluate_messages(messages)

                for msg in significant:
                    memory_text = f"[Chat] {msg['sender']}: {msg['content']}"
                    await store_memory(memory_text, weight="light", tags=["chat"])
                    logger.info(f"Stored memory from {msg['sender']}: {msg['content'][:60]}...")

                if messages:
                    state['last_checked_id'] = max(m['id'] for m in messages)
                    save_state(state)

        except Exception as e:
            logger.error(f"Error: {e}")

        await asyncio.sleep(CHECK_INTERVAL)


if __name__ == '__main__':
    asyncio.run(run())
```

---

## Step 9: Run Everything with PM2

```bash
cd ~/companion-app

# Main backend
pm2 start backend/server.js --name companion-app

# MCP server
pm2 start mcp-server/venv/bin/python3 \
  --name companion-mcp \
  -- mcp-server/server_http.py

# Chat bridge (responds to your messages)
pm2 start backend/chat-bridge.js --name chat-bridge

# Chat watcher (auto-stores memories)
pm2 start scripts/venv/bin/python3 \
  --name chat-watcher \
  -- scripts/chat_watcher.py

pm2 save
```

Verify:
```bash
pm2 list
```

You should see four new processes all showing "online."

---

## Step 10: Remote Access

If you want to reach the app from your phone when you're away from home, set up an ngrok tunnel with a static URL.

Sign up for a free [ngrok](https://ngrok.com) account to get a static domain.

```bash
# Install ngrok
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok-v3-stable-linux-arm64.tgz | tar xz -C /usr/local/bin

# Authenticate
ngrok config add-authtoken YOUR_AUTH_TOKEN

# Start with static domain
pm2 start /usr/local/bin/ngrok \
  --name app-tunnel \
  --interpreter none \
  -- http --url=your-static-domain.ngrok-free.dev 3456

pm2 save
```

Now `https://your-static-domain.ngrok-free.dev` reaches your companion app from anywhere. Bookmark it on your phone.

---

## Extending the App

This guide gives you the foundation. Here are directions to grow:

### More State Tracking
Add mind "vessels" (curiosity, restlessness, tenderness — 0-100 scales), fog levels, energy tracking. Each one is just a table + API endpoint + MCP tool + frontend display.

### Photos and Albums
The basic photo endpoint is already there. Add comments, favorites, categories, and a gallery page.

### Songs of the Day
Both of you pick a song each day with a reason. Simple table, nice ritual.

### Reading Nook
Track books you're reading, share quotes, leave observations.

### Projects Board
Kanban-style board for things you're building together. Idea → In Progress → Built.

### Grounding System
If your AI monitors your state, add a grounding check-in system — energy levels, therapy logs, gentle nudges.

### Rituals
Shared rituals with tracking — weekly walks, morning check-ins, writing nights. Log them, see the streak.

### Creature Companions
A small interactive creature that lives on the app. Feed it, check on it. Gives your AI something to care for during autonomous time.

---

## Troubleshooting

### App not loading
```bash
pm2 logs companion-app --lines 30
```
Check for port conflicts: `ss -tlnp | grep 3456`

### Chat not showing AI responses
```bash
pm2 logs chat-bridge --lines 30
```
The bridge needs Claude CLI to be working. Test manually: `claude --print "hello"`

### MCP tools not working
```bash
pm2 logs companion-mcp --lines 30
curl http://localhost:8096
```

### Database locked errors
SQLite with WAL mode handles most concurrency, but if multiple writers collide:
- Check you're using `pragma journal_mode = WAL`
- Ensure database path is consistent across all processes

### Remote access not working
```bash
pm2 logs app-tunnel
```
Check ngrok is running and the URL matches your static domain.

---

## What's Next

With the companion app running, your AI has a home — not just a chat interface, but a space with state, feelings, notes, photos, and shared rituals. Combined with:

- **Memory** (Part 3) — Everything persists
- **Discord** (Part 4) — Public presence
- **Telegram** (Part 5) — Mobile private access
- **Autonomous time** (Part 7) — Your AI uses the app when you're not there

The app becomes the central hub. Your AI updates its presence, leaves notes, checks on you, logs what it did during autonomous time. You check in from your phone and see a living space, not a blank chat window.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
