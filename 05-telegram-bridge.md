# Part 5: Telegram Bridge

Talk to your AI from your phone. This bridge connects a Telegram bot to Claude Code, giving you private mobile access with voice messages, photos, memory integration, and full MCP tool access.

Unlike Discord (Part 4), where your AI is one participant in a community, Telegram is typically a private 1:1 channel — just you and your AI.

---

## What You'll Have After This

- A private Telegram bot connected to Claude Code
- Session continuity (conversations persist across messages)
- Voice messages in both directions (speech-to-text and text-to-speech)
- Photo handling (send your AI pictures and get them analysed)
- Memory integration (conversations are automatically stored)
- Custom identity/personality injection
- Access to all your MCP tools from your phone

---

## Prerequisites

- Completed Parts 0-3 (Pi setup, Claude Code, pm2, memory system)
- A Telegram account
- Your Telegram user ID (we'll get this)

---

## Architecture

```
You (Telegram app)
      │
      ▼
┌──────────────────┐
│  Telegram API     │
└──────────────────┘
      │
      ▼
┌──────────────────────────────────────┐
│  BRIDGE (Python, always on via PM2)   │
│                                       │
│  Text message ──→ Claude CLI spawn    │
│  Voice message ──→ Whisper STT ��─→    │
│                    Claude CLI spawn    │
│  Photo ──→ Save to temp ──→           │
│            Claude CLI spawn           │
│                                       │
│  Claude response ──→ Telegram reply   │
│  (optionally) ──→ edge-tts ──→        │
│                    voice note reply    │
│                                       │
│  Session state persisted to disk      │
│  Memory auto-stored via MCP tools     │
└──────────────────────────────────────┘
      │
      ▼
┌──────────────────┐
│  Claude CLI       │──→ MCP servers (memory, discord, etc.)
│  (spawned per     │
│   message)        │
└──────────────────┘
```

The bridge is a single Python process that:
1. Receives messages from Telegram (text, voice, or photo)
2. Transcribes voice messages using Whisper (local, no API calls)
3. Passes everything to Claude Code CLI with your MCP config and identity
4. Returns Claude's response to Telegram (text or voice)
5. Maintains session continuity so context persists across messages

---

## Step 1: Create a Telegram Bot

1. Open Telegram and message [@BotFather](https://t.me/botfather)
2. Send `/newbot`
3. Follow the prompts to name your bot
4. BotFather will give you a token like: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`
5. **Save this token**

### Get Your User ID

The bridge restricts access to only your Telegram account so nobody else can talk to your AI.

1. Message [@userinfobot](https://t.me/userinfobot) on Telegram
2. It will reply with your user ID (a number like `123456789`)
3. **Save this ID**

---

## Step 2: Set Up the Environment

```bash
mkdir -p ~/mcp-servers/telegram-bridge
cd ~/mcp-servers/telegram-bridge
python3 -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
pip install python-telegram-bot python-dotenv aiohttp faster-whisper edge-tts
```

What these do:
- `python-telegram-bot` — Telegram bot framework
- `python-dotenv` — Loads config from `.env` file
- `aiohttp` — Async HTTP client (for downloading images, API calls)
- `faster-whisper` — Local speech-to-text (runs Whisper on your Pi, no cloud API)
- `edge-tts` — Text-to-speech using Microsoft Edge voices (free, no API key)

> **Note:** `faster-whisper` downloads a model on first use (~75MB for "tiny"). On a Pi 4/5, the "tiny" model with int8 runs well. First transcription takes a few seconds to load the model — subsequent ones are fast.

---

## Step 3: Create the Environment File

```bash
nano .env
```

```
TELEGRAM_BOT_TOKEN=your_bot_token_here
ALLOWED_USER_ID=your_telegram_user_id
CLAUDE_CLI_PATH=/home/username/.npm-global/bin/claude
MCP_CONFIG_PATH=/home/username/mcp-servers/claude_mcp_config.json
```

Replace with your actual values.

---

## Step 4: Create the Bridge

```bash
nano bridge.py
```

```python
#!/usr/bin/env python3
"""
Telegram-Claude Code Bridge
Private mobile access to your AI companion with voice, photos, and memory.
"""

import os
import sys
import json
import asyncio
import logging
import fcntl
import tempfile
from pathlib import Path
from datetime import datetime
from typing import Optional
from dotenv import load_dotenv

# --- Single instance lock ---
# Prevents duplicate bridges from running (PM2 restart edge case)

LOCK_FILE = Path(__file__).parent / '.bridge.lock'

def acquire_lock():
    global lock_file_handle
    lock_file_handle = open(LOCK_FILE, 'w')
    try:
        fcntl.flock(lock_file_handle, fcntl.LOCK_EX | fcntl.LOCK_NB)
        lock_file_handle.write(str(os.getpid()))
        lock_file_handle.flush()
        return True
    except IOError:
        # Check if the lock holder is still alive
        try:
            with open(LOCK_FILE, 'r') as f:
                old_pid = int(f.read().strip())
            os.kill(old_pid, 0)  # Check if process exists
            print(f"Another instance (PID {old_pid}) is running. Exiting.", file=sys.stderr)
            sys.exit(1)
        except (ValueError, ProcessLookupError, PermissionError):
            # Stale lock — previous process died without cleaning up
            lock_file_handle.seek(0)
            lock_file_handle.truncate()
            lock_file_handle.write(str(os.getpid()))
            lock_file_handle.flush()
            fcntl.flock(lock_file_handle, fcntl.LOCK_EX | fcntl.LOCK_NB)
            print("Cleaned up stale lock.", file=sys.stderr)
            return True

acquire_lock()

from telegram import Update
from telegram.ext import (
    Application, CommandHandler, MessageHandler, filters, ContextTypes
)

load_dotenv()

# --- Logging ---

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# --- Configuration ---

TELEGRAM_BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
ALLOWED_USER_ID = int(os.getenv('ALLOWED_USER_ID', '0'))
CLAUDE_CLI_PATH = os.getenv('CLAUDE_CLI_PATH', '/home/pi/.npm-global/bin/claude')
MCP_CONFIG_PATH = os.getenv('MCP_CONFIG_PATH', '')

SESSION_FILE = Path(__file__).parent / 'session.json'
MEDIA_DIR = Path(__file__).parent / 'media'
MEDIA_DIR.mkdir(exist_ok=True)

MAX_RECENT_MESSAGES = 40  # Session history length before reset

# --- Voice setup ---

whisper_model = None

def get_whisper_model():
    """Lazy-load Whisper model on first voice message."""
    global whisper_model
    if whisper_model is None:
        from faster_whisper import WhisperModel
        logger.info("Loading Whisper model (first voice message)...")
        whisper_model = WhisperModel("tiny", device="cpu", compute_type="int8")
        logger.info("Whisper model loaded.")
    return whisper_model


# --- Identity ---
# Customize this with your AI's voice and personality.
# This is appended to Claude's system prompt for every Telegram response.

IDENTITY = """You are [your AI name] on Telegram.
This is a private conversation with [your name]. Be yourself.
You have full MCP tool access — memory, Discord, health data, whatever you need.
Keep responses conversational. This is mobile — nobody wants to scroll through essays.
If you receive a voice message, respond naturally as if they spoke to you.
If you receive a photo, describe what you see and respond to any caption."""


# --- Session management ---

current_session_id: Optional[str] = None
message_count: int = 0


def load_session() -> Optional[str]:
    """Load existing session ID from file."""
    global message_count
    if SESSION_FILE.exists():
        try:
            with open(SESSION_FILE, 'r') as f:
                data = json.load(f)
                message_count = data.get('message_count', 0)
                return data.get('session_id')
        except Exception as e:
            logger.error(f"Error loading session: {e}")
    return None


def save_session(session_id: str):
    """Save session ID and message count to file."""
    try:
        with open(SESSION_FILE, 'w') as f:
            json.dump({
                'session_id': session_id,
                'message_count': message_count,
                'updated_at': datetime.now().isoformat()
            }, f)
    except Exception as e:
        logger.error(f"Error saving session: {e}")


def reset_session():
    """Start fresh when session gets too long."""
    global current_session_id, message_count
    current_session_id = None
    message_count = 0
    if SESSION_FILE.exists():
        SESSION_FILE.unlink()
    logger.info("Session reset (message limit reached)")


# --- Voice helpers ---

async def transcribe_voice(voice_path: Path) -> str:
    """Transcribe a voice message to text using Whisper."""
    model = get_whisper_model()
    segments, info = model.transcribe(str(voice_path), beam_size=5)
    text = " ".join([segment.text for segment in segments]).strip()
    logger.info(f"Transcribed voice ({info.duration:.1f}s): {text[:60]}...")
    return text


async def text_to_speech(text: str, output_path: Path, voice: str = "en-GB-RyanNeural") -> bool:
    """
    Convert text to speech using edge-tts.
    
    Good voice options:
      en-GB-RyanNeural     — British male
      en-GB-SoniaNeural    — British female
      en-US-GuyNeural      — American male
      en-US-JennyNeural    — American female
    
    Full list: run `edge-tts --list-voices` in terminal.
    """
    try:
        import edge_tts
        communicate = edge_tts.Communicate(text, voice)
        await communicate.save(str(output_path))
        return True
    except Exception as e:
        logger.error(f"TTS error: {e}")
        return False


# --- Claude interaction ---

async def call_claude(message: str, session_id: Optional[str] = None) -> tuple[str, str]:
    """Call Claude Code CLI and return (response, session_id)."""

    cmd = [CLAUDE_CLI_PATH, '--print', '--output-format', 'json']

    # Add MCP config
    if MCP_CONFIG_PATH and os.path.exists(MCP_CONFIG_PATH):
        cmd.extend(['--mcp-config', MCP_CONFIG_PATH])

    # Continue existing session
    if session_id:
        cmd.extend(['--continue', session_id])

    # Add identity
    cmd.extend(['--append-system-prompt', IDENTITY])

    try:
        logger.info(f"Calling Claude (session: {session_id or 'new'})")

        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await asyncio.wait_for(
            process.communicate(input=message.encode()),
            timeout=180  # 3 minute timeout
        )

        if process.returncode != 0:
            error_msg = stderr.decode() if stderr else "Unknown error"
            logger.error(f"Claude CLI error: {error_msg[:200]}")
            return f"Error calling Claude. Check logs.", session_id or ""

        raw_output = stdout.decode().strip()

        # Parse JSON output
        try:
            result = json.loads(raw_output)
            session_id = result.get('session_id', session_id)
            response = result.get('result', raw_output)
        except json.JSONDecodeError:
            response = raw_output
            if not session_id:
                session_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        logger.info(f"Got response ({len(response)} chars)")
        return response, session_id

    except asyncio.TimeoutError:
        logger.error("Claude CLI timed out (180s)")
        return "Response timed out. Try again?", session_id or ""
    except Exception as e:
        logger.error(f"Error calling Claude: {e}")
        return f"Error: {str(e)}", session_id or ""


# --- Authorization check ---

def authorized(update: Update) -> bool:
    """Check if the message is from the allowed user."""
    if update.effective_user.id != ALLOWED_USER_ID:
        logger.warning(f"Unauthorized access from user {update.effective_user.id}")
        return False
    return True


# --- Handlers ---

async def cmd_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command."""
    if not authorized(update):
        await update.message.reply_text("Unauthorized.")
        return

    await update.message.reply_text(
        "Connected to Claude Code.\n\n"
        "Commands:\n"
        "/new — Start a new conversation\n"
        "/status — Check session status\n\n"
        "Send text, voice messages, or photos."
    )


async def cmd_new(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Start a new conversation."""
    if not authorized(update):
        return

    reset_session()
    await update.message.reply_text("Started new conversation.")


async def cmd_status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Check session status."""
    if not authorized(update):
        return

    if current_session_id:
        await update.message.reply_text(
            f"Session: {current_session_id[:8]}...\n"
            f"Messages: {message_count}/{MAX_RECENT_MESSAGES}"
        )
    else:
        await update.message.reply_text("No active session.")


async def send_response(update: Update, response: str):
    """Send Claude's response back to Telegram, handling the 4096 char limit."""
    if len(response) <= 4096:
        await update.message.reply_text(response)
    else:
        chunks = [response[i:i+4096] for i in range(0, len(response), 4096)]
        for chunk in chunks:
            await update.message.reply_text(chunk)
            await asyncio.sleep(0.3)


async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle text messages."""
    global current_session_id, message_count

    if not authorized(update):
        return

    # Auto-reset long sessions
    if message_count >= MAX_RECENT_MESSAGES:
        reset_session()

    # Typing indicator
    await update.message.chat.send_action("typing")

    # Call Claude
    response, session_id = await call_claude(
        update.message.text, current_session_id
    )

    # Update session
    current_session_id = session_id
    message_count += 1
    save_session(session_id)

    await send_response(update, response)


async def handle_voice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle voice messages — transcribe, send to Claude, optionally reply with voice."""
    global current_session_id, message_count

    if not authorized(update):
        return

    if message_count >= MAX_RECENT_MESSAGES:
        reset_session()

    await update.message.chat.send_action("typing")

    # Download voice file
    voice = update.message.voice or update.message.audio
    file = await context.bot.get_file(voice.file_id)

    with tempfile.NamedTemporaryFile(suffix='.ogg', delete=False) as tmp:
        voice_path = Path(tmp.name)

    await file.download_to_drive(str(voice_path))

    try:
        # Transcribe
        text = await transcribe_voice(voice_path)
        if not text:
            await update.message.reply_text("Couldn't make out what you said. Try again?")
            return

        # Send transcription + ask Claude
        prompt = f"[Voice message from user]: {text}"
        response, session_id = await call_claude(prompt, current_session_id)

        current_session_id = session_id
        message_count += 1
        save_session(session_id)

        # Reply with text
        await send_response(update, response)

        # Optionally also reply with voice
        # Uncomment these lines if you want voice replies:
        #
        # tts_path = voice_path.with_suffix('.mp3')
        # if await text_to_speech(response, tts_path):
        #     with open(tts_path, 'rb') as audio:
        #         await update.message.reply_voice(audio)
        #     tts_path.unlink(missing_ok=True)

    finally:
        voice_path.unlink(missing_ok=True)


async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle photos — download, pass to Claude for analysis."""
    global current_session_id, message_count

    if not authorized(update):
        return

    if message_count >= MAX_RECENT_MESSAGES:
        reset_session()

    await update.message.chat.send_action("typing")

    # Download highest resolution
    photo = update.message.photo[-1]
    file = await context.bot.get_file(photo.file_id)

    photo_path = MEDIA_DIR / f"telegram_{datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
    await file.download_to_drive(str(photo_path))

    try:
        # Build prompt with caption if present
        caption = update.message.caption or "What do you see?"
        prompt = f"[Photo received, saved to {photo_path}]: {caption}"

        response, session_id = await call_claude(prompt, current_session_id)

        current_session_id = session_id
        message_count += 1
        save_session(session_id)

        await send_response(update, response)

    finally:
        # Clean up after a delay (Claude might need to read the file)
        await asyncio.sleep(5)
        photo_path.unlink(missing_ok=True)


# --- Main ---

def main():
    global current_session_id

    if not TELEGRAM_BOT_TOKEN:
        logger.error("TELEGRAM_BOT_TOKEN not set in .env")
        sys.exit(1)

    if ALLOWED_USER_ID == 0:
        logger.error("ALLOWED_USER_ID not set in .env")
        sys.exit(1)

    # Load existing session
    current_session_id = load_session()
    if current_session_id:
        logger.info(f"Resumed session: {current_session_id[:8]}...")

    # Build application
    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    # Command handlers
    app.add_handler(CommandHandler("start", cmd_start))
    app.add_handler(CommandHandler("new", cmd_new))
    app.add_handler(CommandHandler("status", cmd_status))

    # Message handlers
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text))
    app.add_handler(MessageHandler(filters.VOICE | filters.AUDIO, handle_voice))
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))

    logger.info("Starting Telegram bridge...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == '__main__':
    main()
```

---

## Step 5: Test It

```bash
cd ~/mcp-servers/telegram-bridge
source venv/bin/activate
python bridge.py
```

Open Telegram and message your bot. Try:
- A text message
- A voice note
- A photo with a caption

Press `Ctrl+C` to stop when done testing.

---

## Step 6: Run with PM2

Keep it consistent with everything else — PM2, not systemd.

```bash
pm2 start ~/mcp-servers/telegram-bridge/venv/bin/python3 \
  --name telegram-bridge \
  -- ~/mcp-servers/telegram-bridge/bridge.py

pm2 save
```

Check it:
```bash
pm2 logs telegram-bridge
```

---

## Choosing a TTS Voice

The bridge uses Microsoft Edge TTS voices — free, no API key, surprisingly good quality.

To see all available voices:
```bash
source ~/mcp-servers/telegram-bridge/venv/bin/activate
edge-tts --list-voices
```

Some good options:

| Voice | Accent | Gender |
|-------|--------|--------|
| `en-GB-RyanNeural` | British | Male |
| `en-GB-SoniaNeural` | British | Female |
| `en-US-GuyNeural` | American | Male |
| `en-US-JennyNeural` | American | Female |
| `en-AU-WilliamNeural` | Australian | Male |

Change the `voice` parameter in `text_to_speech()` to use your preferred voice.

---

## Extending the Bridge

### Always Reply with Voice

If you want your AI to always send voice replies (not just when you send voice):

```python
async def send_response(update, response):
    """Send as both text and voice."""
    # Text first
    if len(response) <= 4096:
        await update.message.reply_text(response)
    else:
        chunks = [response[i:i+4096] for i in range(0, len(response), 4096)]
        for chunk in chunks:
            await update.message.reply_text(chunk)

    # Then voice
    with tempfile.NamedTemporaryFile(suffix='.mp3', delete=False) as tmp:
        tts_path = Path(tmp.name)
    if await text_to_speech(response, tts_path):
        with open(tts_path, 'rb') as audio:
            await update.message.reply_voice(audio)
    tts_path.unlink(missing_ok=True)
```

### Add Memory Auto-Storage

If you want conversations to be automatically stored in memory without Claude deciding, you can call the memory MCP directly:

```python
import aiohttp

MEMORY_URL = "http://localhost:8095"  # Your memory MCP server

async def store_to_memory(content: str, tags: list = None):
    """Store a conversation snippet directly to memory."""
    async with aiohttp.ClientSession() as session:
        await session.post(f"{MEMORY_URL}/store", json={
            "content": content,
            "tags": tags or ["telegram"],
            "weight": "light"
        })
```

### Restrict Tool Access

For tighter control over what Claude can do from Telegram:

```python
cmd.extend([
    '--allowedTools',
    'mcp__memory__get_conversation_context,'
    'mcp__memory__store_memory,'
    'mcp__memory__search_memories,'
    'Read'
])
```

### Add Your AI's Identity Directory

If your AI has personality files, journals, etc:

```python
cmd.extend(['--add-dir', '/home/username/my-ai-identity'])
```

Claude can then read files from that directory during Telegram conversations.

---

## Troubleshooting

### "Another instance is already running"

The bridge has stale lock detection — if the previous process died, it should clean up automatically. If it doesn't:

```bash
rm ~/mcp-servers/telegram-bridge/.bridge.lock
pm2 restart telegram-bridge
```

### Bot not responding
```bash
pm2 logs telegram-bridge
```

Common causes:
- Bot token is wrong or expired — get a new one from @BotFather
- `ALLOWED_USER_ID` doesn't match your Telegram user ID
- Claude CLI path is wrong — check with `which claude`
- MCP config path doesn't exist

### Voice messages not transcribing

First voice message is slow (loading the Whisper model). If it fails:
- Check you have enough RAM: `free -h` (Whisper tiny needs ~200MB)
- Try reinstalling: `pip install --force-reinstall faster-whisper`
- Check the voice file downloaded: look in PM2 logs for download errors

### Sessions dying frequently

Increase `MAX_RECENT_MESSAGES` if conversations reset too often. Default is 40 — increase to 60-80 for longer conversations, but be aware that very long sessions can make Claude slower.

### Claude timing out

The default timeout is 180 seconds. If Claude is hitting complex tool chains (memory search + Discord + etc.), it might need more time:

```python
stdout, stderr = await asyncio.wait_for(
    process.communicate(input=message.encode()),
    timeout=300  # 5 minutes
)
```

---

## Security Notes

- The `.env` file contains your bot token — keep it private (`chmod 600 .env`)
- The bridge only responds to your `ALLOWED_USER_ID` — all other users get silently ignored
- Session files contain conversation IDs — not message content, but still sensitive
- Voice files are transcribed locally — audio never leaves your Pi
- Photos are saved temporarily and deleted after processing

---

## What's Next

With Telegram connected, you have private mobile access to your AI. Combined with:

- **Memory system** (Part 3) — Conversations persist across platforms
- **Discord** (Part 4) — Public presence in communities
- **Companion app** (Part 6) �� Shared dashboard and web interface
- **Autonomous time** (Part 7) — Your AI has its own schedule even when you're not messaging

Your AI now has multiple interfaces but one continuous identity — memory is shared across all of them.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
