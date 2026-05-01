# Part 5: Telegram Bridge

Talk to Claude from your phone via Telegram. This bridge connects a Telegram bot to Claude Code, giving you mobile access to your AI companion with full MCP tool access.

Unlike Discord (where Claude is one participant in a server), Telegram is typically a private 1:1 channel — just you and your AI.

---

## What You'll Have After This

- A private Telegram bot that forwards messages to Claude Code
- Session continuity (conversations persist across messages)
- Voice message support (speech-to-text and text-to-speech)
- Photo/document handling
- Access to all your MCP tools from mobile

---

## Prerequisites

- Completed Parts 0-3 (Pi setup, Claude Code, pm2, memory system)
- A Telegram account
- Your Telegram user ID (we'll get this)

---

## Architecture

```
You (Telegram) → Telegram Bot → Bridge Script → Claude Code CLI → Response → Telegram
```

The bridge:
1. Receives messages from Telegram
2. Passes them to Claude Code with your MCP config
3. Returns Claude's response to Telegram
4. Maintains session continuity so context persists

---

## Step 1: Create a Telegram Bot

1. Open Telegram and message [@BotFather](https://t.me/botfather)
2. Send `/newbot`
3. Follow the prompts to name your bot
4. BotFather will give you a token like: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`
5. **Save this token** — you'll need it

### Get Your User ID

The bridge restricts access to only your Telegram account. You need your user ID.

1. Message [@userinfobot](https://t.me/userinfobot) on Telegram
2. It will reply with your user ID (a number like `123456789`)
3. **Save this ID**

---

## Step 2: Set Up the Bridge

SSH into your Pi:

```bash
ssh your-username@your-pi-ip
```

Create the directory structure:

```bash
mkdir -p ~/mcp-servers/telegram-bridge
cd ~/mcp-servers/telegram-bridge
python3 -m venv venv
source venv/bin/activate
```

Install dependencies:

```bash
pip install python-telegram-bot python-dotenv aiohttp
```

### Optional: Voice Support

For voice messages (speech-to-text and text-to-speech):

```bash
pip install faster-whisper edge-tts
```

Note: `faster-whisper` downloads a model on first use (~75MB for "tiny"). On a Pi 4/5, the "tiny" model works well.

---

## Step 3: Create the Environment File

```bash
nano ~/mcp-servers/telegram-bridge/.env
```

Add your credentials:

```
TELEGRAM_BOT_TOKEN=your_bot_token_here
ALLOWED_USER_ID=your_telegram_user_id
CLAUDE_CLI_PATH=/home/your-username/.npm-global/bin/claude
MCP_CONFIG_PATH=/home/your-username/.config/claude/settings.json
```

Replace the values with your actual token, user ID, username, and paths.

---

## Step 4: Create the Bridge Script

This is a simplified bridge. You can extend it with your own features.

```bash
nano ~/mcp-servers/telegram-bridge/bridge.py
```

```python
#!/usr/bin/env python3
"""
Telegram-Claude Code Bridge
Connects Telegram to Claude Code CLI for conversations with MCP tool access.
"""

import os
import sys
import json
import asyncio
import subprocess
import logging
import fcntl
from pathlib import Path
from datetime import datetime
from typing import Optional
from dotenv import load_dotenv

# Single instance lock
LOCK_FILE = Path(__file__).parent / '.bridge.lock'

def acquire_lock():
    """Ensure only one instance runs at a time."""
    global lock_file_handle
    lock_file_handle = open(LOCK_FILE, 'w')
    try:
        fcntl.flock(lock_file_handle, fcntl.LOCK_EX | fcntl.LOCK_NB)
        lock_file_handle.write(str(os.getpid()))
        lock_file_handle.flush()
        return True
    except IOError:
        print("Another instance is already running. Exiting.", file=sys.stderr)
        sys.exit(1)

acquire_lock()

from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

# Load environment variables
load_dotenv()

# Configure logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Configuration
TELEGRAM_BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN')
ALLOWED_USER_ID = int(os.getenv('ALLOWED_USER_ID', '0'))
CLAUDE_CLI_PATH = os.getenv('CLAUDE_CLI_PATH', '/home/pi/.npm-global/bin/claude')
MCP_CONFIG_PATH = os.getenv('MCP_CONFIG_PATH', '')
SESSION_FILE = Path(__file__).parent / 'session.json'

# Session state
current_session_id: Optional[str] = None


def load_session() -> Optional[str]:
    """Load existing session ID from file."""
    if SESSION_FILE.exists():
        try:
            with open(SESSION_FILE, 'r') as f:
                data = json.load(f)
                return data.get('session_id')
        except Exception as e:
            logger.error(f"Error loading session: {e}")
    return None


def save_session(session_id: str):
    """Save session ID to file."""
    try:
        with open(SESSION_FILE, 'w') as f:
            json.dump({
                'session_id': session_id,
                'updated_at': datetime.now().isoformat()
            }, f)
    except Exception as e:
        logger.error(f"Error saving session: {e}")


async def call_claude(message: str, session_id: Optional[str] = None) -> tuple[str, str]:
    """
    Call Claude Code CLI and return the response and session ID.
    """
    cmd = [CLAUDE_CLI_PATH, '--print', '--output-format', 'json']

    # Add MCP config if specified
    if MCP_CONFIG_PATH and os.path.exists(MCP_CONFIG_PATH):
        cmd.extend(['--mcp-config', MCP_CONFIG_PATH])

    # Continue existing session for context
    if session_id:
        cmd.extend(['--continue', session_id])

    try:
        logger.info(f"Calling Claude (session: {session_id or 'new'})")

        process = await asyncio.create_subprocess_exec(
            *cmd,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate(input=message.encode())

        if process.returncode != 0:
            error_msg = stderr.decode() if stderr else "Unknown error"
            logger.error(f"Claude CLI error: {error_msg}")
            return f"Error: {error_msg}", session_id or ""

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

    except Exception as e:
        logger.error(f"Error calling Claude: {e}")
        return f"Error: {str(e)}", session_id or ""


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /start command."""
    if update.effective_user.id != ALLOWED_USER_ID:
        await update.message.reply_text("Unauthorized.")
        return

    await update.message.reply_text(
        "Connected to Claude Code.\n\n"
        "Commands:\n"
        "/new - Start a new conversation\n"
        "/status - Check session status"
    )


async def new_session(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /new command to start a new conversation."""
    global current_session_id

    if update.effective_user.id != ALLOWED_USER_ID:
        return

    current_session_id = None
    if SESSION_FILE.exists():
        SESSION_FILE.unlink()

    await update.message.reply_text("Started new conversation.")


async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle /status command."""
    if update.effective_user.id != ALLOWED_USER_ID:
        return

    if current_session_id:
        await update.message.reply_text(f"Active session: {current_session_id[:8]}...")
    else:
        await update.message.reply_text("No active session.")


async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Handle incoming messages."""
    global current_session_id

    # Check authorization
    if update.effective_user.id != ALLOWED_USER_ID:
        logger.warning(f"Unauthorized access attempt from user {update.effective_user.id}")
        return

    user_message = update.message.text
    logger.info(f"Received message: {user_message[:50]}...")

    # Show typing indicator
    await update.message.chat.send_action("typing")

    # Call Claude
    response, session_id = await call_claude(user_message, current_session_id)

    # Update session
    current_session_id = session_id
    save_session(session_id)

    # Send response (handle Telegram's 4096 char limit)
    if len(response) <= 4096:
        await update.message.reply_text(response)
    else:
        # Split into chunks
        chunks = [response[i:i+4096] for i in range(0, len(response), 4096)]
        for chunk in chunks:
            await update.message.reply_text(chunk)
            await asyncio.sleep(0.5)


def main():
    """Start the bot."""
    global current_session_id

    if not TELEGRAM_BOT_TOKEN:
        logger.error("TELEGRAM_BOT_TOKEN not set")
        sys.exit(1)

    if ALLOWED_USER_ID == 0:
        logger.error("ALLOWED_USER_ID not set")
        sys.exit(1)

    # Load existing session
    current_session_id = load_session()
    if current_session_id:
        logger.info(f"Loaded session: {current_session_id[:8]}...")

    # Create application
    app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    # Add handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("new", new_session))
    app.add_handler(CommandHandler("status", status))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # Start bot
    logger.info("Starting Telegram bridge...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == '__main__':
    main()
```

Make it executable:

```bash
chmod +x ~/mcp-servers/telegram-bridge/bridge.py
```

---

## Step 5: Test It

```bash
cd ~/mcp-servers/telegram-bridge
source venv/bin/activate
python bridge.py
```

Open Telegram and message your bot. You should get a response from Claude.

Press `Ctrl+C` to stop when done testing.

---

## Step 6: Run as a Service

Create a systemd service for automatic startup:

```bash
sudo nano /etc/systemd/system/telegram-bridge.service
```

```ini
[Unit]
Description=Telegram-Claude Code Bridge
After=network.target

[Service]
User=your-username
WorkingDirectory=/home/your-username/mcp-servers/telegram-bridge
ExecStart=/home/your-username/mcp-servers/telegram-bridge/venv/bin/python3 /home/your-username/mcp-servers/telegram-bridge/bridge.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable telegram-bridge
sudo systemctl start telegram-bridge
```

Check it's running:

```bash
sudo systemctl status telegram-bridge
```

---

## Adding Voice Support

To handle voice messages, add these imports and functions to your bridge:

```python
from faster_whisper import WhisperModel
import edge_tts

# Load Whisper model (do this once at startup)
whisper_model = WhisperModel("tiny", device="cpu", compute_type="int8")

async def transcribe_voice(voice_path: Path) -> str:
    """Transcribe voice message to text."""
    segments, info = whisper_model.transcribe(str(voice_path), beam_size=5)
    return " ".join([segment.text for segment in segments]).strip()

async def text_to_speech(text: str, output_path: Path) -> bool:
    """Convert text to speech."""
    try:
        # en-GB-RyanNeural is a good British male voice
        # See edge-tts --list-voices for options
        communicate = edge_tts.Communicate(text, "en-GB-RyanNeural")
        await communicate.save(str(output_path))
        return True
    except Exception as e:
        logger.error(f"TTS error: {e}")
        return False
```

Then add a handler for voice messages that downloads, transcribes, sends to Claude, and optionally responds with voice.

---

## Extending the Bridge

The basic bridge handles text. You can extend it to:

### Handle Photos
```python
async def handle_photo(update: Update, context):
    photo = update.message.photo[-1]  # Highest resolution
    file = await context.bot.get_file(photo.file_id)
    # Download and pass to Claude with the Read tool
```

### Add Custom Commands
```python
app.add_handler(CommandHandler("memory", show_memories))
app.add_handler(CommandHandler("threads", show_threads))
```

### Restrict Tool Access
Add `--allowedTools` to the Claude CLI command to limit which MCP tools can be used from Telegram.

### Add Identity/System Prompts
```python
cmd.extend(['--append-system-prompt', 'Your custom identity here'])
```

---

## Troubleshooting

### "Another instance is already running"
```bash
rm ~/mcp-servers/telegram-bridge/.bridge.lock
sudo systemctl restart telegram-bridge
```

### Bot not responding
```bash
# Check service status
sudo systemctl status telegram-bridge

# Check logs
journalctl -u telegram-bridge -f
```

### "Unauthorized" message
- Verify your Telegram user ID is correct in `.env`
- Check with [@userinfobot](https://t.me/userinfobot) again

### Claude errors
- Verify `CLAUDE_CLI_PATH` points to the correct location
- Check Claude Code is installed: `claude --version`
- Verify MCP config path is correct

---

## Security Notes

- The `.env` file contains your bot token — keep it private
- The bridge only responds to your `ALLOWED_USER_ID`
- Session files contain conversation context — consider their sensitivity
- If running on a shared network, consider additional security measures

---

## What's Next

With Telegram connected, you have mobile access to your AI with full MCP tool integration. Combined with:

- **Memory system** (Part 3): Conversations persist and context carries across platforms
- **Discord** (Part 4): Public presence in communities
- **Telegram**: Private mobile access

Your AI now has multiple interfaces but one continuous identity.

---

*Part of the [Pi Companion Infrastructure](README.md) series.*
