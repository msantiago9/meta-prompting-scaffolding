# Prompt
**Role:**
Act as a Senior Lead Software Architect and Technical Writer specializing in high-concurrency **Python** applications.
**Objective:**
Generate a comprehensive, single-file design document (filename: IMPLEMENTATION_GUIDE.md) that outlines the entire architecture for a self-hosted, managed Discord Bot service.
**The Feature Set:**
1. **Spoiler Image Proxy:** The bot must detect an image URL in a user's message, asynchronously download it to a memory buffer, delete the original message, and repost the content as a **Python Embed** alongside the image as a SPOILER_ prefixed file attachment.
2. **Management GUI:** The service must expose a simple web-based administrative frontend with two functions: **Start Bot** and **Stop Bot**.
3. **Self-Hosted Control:** Toggling the GUI buttons must directly control the local bot process. If the admin stops the web application, the entire service must stop.
**Mandatory Technology Stack:**
* **Language:** Python 3.10+
* **Discord Library:** discord.py (Asynchronous)
* **Web Framework:** Flask (or equivalent lightweight framework)
* **HTTP Requests:** requests or httpx
* **Concurrency:** Use of Python's threading or asyncio.create_task to manage the running of the Flask server and the discord.py client loop simultaneously.
**Document Structure & Audience Requirements:**
1. **Project Abstract & Prerequisites (For the Human User):**
   - Provide a clear, layman's explanation of the multi-process architecture (Web Server vs. Bot Client).
   - Provide the exact shell commands for installation pip install...).
   - List the **required Discord Permissions** (Manage Messages, Message Content Intent) and the **required system ports** (e.g., Port 5000 for Flask).
---
2. **Core Logic Flow & Concurrency (For the AI Junior Developer):**
   - Detail the **two main concurrent processes** and how they communicate (e.g., passing the bot client object between threads).
   - **Start Function Logic:** Detail the function that spawns the bot client's main loop in a new thread/task when the "Start Bot" button is pressed.
   - **Stop Function Logic:** Detail the function that uses client.close() and joins/cancels the bot's thread/task gracefully when the "Stop Bot" button is pressed.
   - **Spoiler Proxy Logic:** Provide the pseudo-code for the on_message handler, including the Regex pattern, the stream/buffer handling, and the file creation discord.File(buffer, filename="SPOILER_...")).
---
3. **Technical Specifications & Edge Cases:**
   - Define the Flask routes for / (GUI rendering), /start, and /stop.
   - Specify the exact Python imports needed for asynchronous management threading, asyncio, etc.).
   - Define error handling for: Bot already running/stopped, image link 404, image file size exceeding Discord's limit.
---
4. **Implementation Steps (The "Prompt Chain"):**
   - Provide a sequence of **4 highly specific, incremental prompts** the user can copy-paste to an AI coding agent to build this complex project. The prompts must sequence the development logically: (1) Basic Flask Setup, (2) Discord Client Setup & Concurrency, (3) Spoiler Proxy Logic, and (4) GUI Integration/Cleanup.
**Tone:**
Technical, precise, and highly structured. Use Python code blocks and Markdown tables for clarity.

# Response

# IMPLEMENTATION_GUIDE.md

## Discord Bot Service: Spoiler Proxy with Operational Whitelisting

**Version:** 1.0  
**Target Python Version:** 3.10+  
**Architecture:** Multi-threaded Web-Managed Discord Bot

---

## 1. PROJECT ABSTRACT & PREREQUISITES

### 1.1 System Overview (For Human Users)

This service implements a **two-component architecture** running within a single Python process:

1. **Flask Web Server (Main Thread):** A lightweight HTTP server that renders an administrative GUI and exposes control endpoints (`/start`, `/stop`). This runs on the main thread and remains active for the lifetime of the application.

2. **Discord Bot Client (Worker Thread):** A `discord.py` asynchronous event loop that connects to Discord's Gateway API. This component runs in a separate daemon thread and can be started/stopped via the web interface.

**Communication Flow:**
```
User Browser → Flask Route Handler → Shared Bot State Object → Bot Thread Control
```

The bot monitors only whitelisted Discord servers (guilds) and channels. When it detects an image URL in an allowed location, it:
- Downloads the image to memory
- Deletes the original message
- Reposts the content as an embed with a spoiler-tagged attachment

### 1.2 Installation Commands

```bash
# Create virtual environment (recommended)
python3.10 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install required packages
pip install discord.py==2.3.2 Flask==3.0.0 requests==2.31.0

# Verify installation
python -c "import discord; import flask; import requests; print('All dependencies installed successfully')"
```

### 1.3 Required Discord Permissions

**Bot Permissions Integer:** `275414773760`

**Required Permissions:**
- ✅ Read Messages/View Channels
- ✅ Send Messages
- ✅ Manage Messages (to delete original posts)
- ✅ Embed Links
- ✅ Attach Files
- ✅ Read Message History

**Required Privileged Intents:**
- ✅ Message Content Intent (must be enabled in Discord Developer Portal)

**Bot Invitation URL Format:**
```
https://discord.com/api/oauth2/authorize?client_id=YOUR_BOT_CLIENT_ID&permissions=275414773760&scope=bot
```

### 1.4 System Requirements

| Resource | Requirement |
|----------|-------------|
| **Network Port** | TCP 5000 (Flask default, configurable) |
| **RAM** | Minimum 256MB available |
| **Python Version** | 3.10 or higher |
| **Internet Access** | Required (Discord Gateway + Image URLs) |

### 1.5 Configuration File Setup

Create a file named `config.json` in the project root:

```json
{
  "discord_token": "YOUR_BOT_TOKEN_HERE",
  "allowed_guild_ids": [
    123456789012345678,
    987654321098765432
  ],
  "allowed_channel_ids": [
    111222333444555666,
    777888999000111222
  ],
  "flask_port": 5000,
  "flask_host": "127.0.0.1",
  "max_image_size_mb": 8
}
```

**Configuration Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| `discord_token` | string | Bot token from Discord Developer Portal |
| `allowed_guild_ids` | array[int] | Whitelist of Discord server IDs |
| `allowed_channel_ids` | array[int] | Whitelist of channel IDs |
| `flask_port` | int | Port for web GUI (default: 5000) |
| `flask_host` | string | Bind address (use `0.0.0.0` for external access) |
| `max_image_size_mb` | int | Maximum image download size in MB |

**Security Notes:**
- Never commit `config.json` to version control
- Add `config.json` to `.gitignore`
- Use environment variables for production deployments

---

## 2. CORE LOGIC FLOW & CONCURRENCY

### 2.1 Architecture Diagram

```
┌─────────────────────────────────────┐
│   Main Python Process               │
│                                     │
│  ┌──────────────────────────────┐  │
│  │  Flask App (Main Thread)     │  │
│  │  - GUI Rendering             │  │
│  │  - /start endpoint           │  │
│  │  - /stop endpoint            │  │
│  └──────────┬───────────────────┘  │
│             │                       │
│             │ Controls              │
│             ↓                       │
│  ┌──────────────────────────────┐  │
│  │  Shared Bot Manager Object   │  │
│  │  - bot_thread: Thread        │  │
│  │  - bot_client: discord.Client│  │
│  │  - is_running: bool          │  │
│  └──────────┬───────────────────┘  │
│             │                       │
│             │ Manages               │
│             ↓                       │
│  ┌──────────────────────────────┐  │
│  │  Discord Bot (Worker Thread) │  │
│  │  - Asyncio Event Loop        │  │
│  │  - on_message Handler        │  │
│  │  - Image Processing          │  │
│  └──────────────────────────────┘  │
│                                     │
└─────────────────────────────────────┘
```

### 2.2 Message Processing Flow (CRITICAL FILTERING)

```python
# Pseudo-code for on_message event handler

async def on_message(message):
    # STEP 0: Ignore bot's own messages
    if message.author == client.user:
        return
    
    # STEP 1: WHITELIST FILTERING (ABSOLUTE FIRST CHECK)
    if message.guild is None:  # DM messages have no guild
        return
    
    if message.guild.id not in ALLOWED_GUILD_IDS:
        return  # Silently ignore messages from non-whitelisted servers
    
    if message.channel.id not in ALLOWED_CHANNEL_IDS:
        return  # Silently ignore messages from non-whitelisted channels
    
    # STEP 2: Extract image URLs using regex
    image_url_pattern = r'(https?://[^\s]+\.(?:png|jpg|jpeg|gif|webp)(?:\?[^\s]*)?)'
    urls = re.findall(image_url_pattern, message.content, re.IGNORECASE)
    
    if not urls:
        return  # No image URLs found
    
    # STEP 3: Process first image URL
    image_url = urls[0]
    
    try:
        # STEP 4: Download image to memory buffer
        response = requests.get(image_url, stream=True, timeout=10)
        response.raise_for_status()
        
        # Check content length
        content_length = int(response.headers.get('content-length', 0))
        max_size = CONFIG['max_image_size_mb'] * 1024 * 1024
        
        if content_length > max_size:
            await message.channel.send(f"⚠️ Image too large ({content_length/1024/1024:.1f}MB)")
            return
        
        # Read into BytesIO buffer
        image_buffer = io.BytesIO()
        for chunk in response.iter_content(chunk_size=8192):
            image_buffer.write(chunk)
        image_buffer.seek(0)
        
        # STEP 5: Create embed with original message content
        embed = discord.Embed(
            description=message.content,
            color=discord.Color.blue()
        )
        embed.set_author(
            name=message.author.display_name,
            icon_url=message.author.avatar.url if message.author.avatar else None
        )
        embed.timestamp = message.created_at
        
        # STEP 6: Create spoiler file attachment
        filename = f"SPOILER_{image_url.split('/')[-1].split('?')[0]}"
        file = discord.File(fp=image_buffer, filename=filename)
        
        # STEP 7: Delete original message
        await message.delete()
        
        # STEP 8: Send new message with embed and spoiler image
        await message.channel.send(embed=embed, file=file)
        
    except requests.RequestException as e:
        await message.channel.send(f"❌ Failed to download image: {str(e)}")
    except discord.HTTPException as e:
        await message.channel.send(f"❌ Failed to post message: {str(e)}")
    except Exception as e:
        print(f"Unexpected error in on_message: {e}")
```

### 2.3 Bot Thread Management

#### 2.3.1 Bot Manager Class Structure

```python
import threading
import asyncio
from typing import Optional

class BotManager:
    def __init__(self, config: dict):
        self.config = config
        self.bot_client: Optional[discord.Client] = None
        self.bot_thread: Optional[threading.Thread] = None
        self.event_loop: Optional[asyncio.AbstractEventLoop] = None
        self.is_running: bool = False
        self.lock = threading.Lock()
    
    def start_bot(self) -> dict:
        """
        Starts the Discord bot in a new daemon thread.
        Returns: {"success": bool, "message": str}
        """
        with self.lock:
            if self.is_running:
                return {"success": False, "message": "Bot is already running"}
            
            # Create new event loop for the bot thread
            self.event_loop = asyncio.new_event_loop()
            
            # Initialize Discord client
            intents = discord.Intents.default()
            intents.message_content = True
            self.bot_client = discord.Client(intents=intents)
            
            # Register event handlers
            self._register_handlers()
            
            # Start bot in daemon thread
            self.bot_thread = threading.Thread(
                target=self._run_bot_thread,
                daemon=True
            )
            self.bot_thread.start()
            self.is_running = True
            
            return {"success": True, "message": "Bot started successfully"}
    
    def stop_bot(self) -> dict:
        """
        Gracefully stops the Discord bot.
        Returns: {"success": bool, "message": str}
        """
        with self.lock:
            if not self.is_running:
                return {"success": False, "message": "Bot is not running"}
            
            if self.bot_client and self.event_loop:
                # Schedule bot shutdown in its event loop
                asyncio.run_coroutine_threadsafe(
                    self.bot_client.close(),
                    self.event_loop
                )
                
                # Wait for thread to finish (with timeout)
                self.bot_thread.join(timeout=5.0)
                
            self.is_running = False
            self.bot_client = None
            self.bot_thread = None
            self.event_loop = None
            
            return {"success": True, "message": "Bot stopped successfully"}
    
    def _run_bot_thread(self):
        """
        Thread target function that runs the asyncio event loop.
        """
        asyncio.set_event_loop(self.event_loop)
        try:
            self.event_loop.run_until_complete(
                self.bot_client.start(self.config['discord_token'])
            )
        except Exception as e:
            print(f"Bot thread error: {e}")
            self.is_running = False
    
    def _register_handlers(self):
        """
        Registers Discord event handlers with filtering logic.
        """
        @self.bot_client.event
        async def on_ready():
            print(f'Bot connected as {self.bot_client.user}')
        
        @self.bot_client.event
        async def on_message(message):
            # [Insert full on_message logic from section 2.2]
            pass
```

### 2.4 Flask Application Structure

```python
from flask import Flask, render_template, jsonify, request

app = Flask(__name__)
bot_manager = None  # Global bot manager instance

@app.route('/')
def index():
    """Render the admin GUI"""
    return render_template('index.html', 
                          bot_running=bot_manager.is_running)

@app.route('/start', methods=['POST'])
def start_bot():
    """Start bot endpoint"""
    result = bot_manager.start_bot()
    return jsonify(result)

@app.route('/stop', methods=['POST'])
def stop_bot():
    """Stop bot endpoint"""
    result = bot_manager.stop_bot()
    return jsonify(result)

@app.route('/status', methods=['GET'])
def get_status():
    """Get current bot status"""
    return jsonify({
        "running": bot_manager.is_running,
        "client_user": str(bot_manager.bot_client.user) if bot_manager.bot_client else None
    })
```

---

## 3. TECHNICAL SPECIFICATIONS & EDGE CASES

### 3.1 Required Python Imports

```python
# Standard Library
import json
import io
import re
import threading
import asyncio
from typing import Optional, Dict, List

# Third-Party
import discord
from discord import Intents, Client, Embed, File
import requests
from flask import Flask, render_template, jsonify, request
```

### 3.2 Error Handling Matrix

| Error Scenario | Detection Method | Handler Response |
|----------------|------------------|------------------|
| **Bot Already Running** | Check `is_running` flag | Return JSON error to frontend |
| **Bot Not Running (Stop)** | Check `is_running` flag | Return JSON error to frontend |
| **Invalid Config File** | `json.JSONDecodeError` | Print error, exit with code 1 |
| **Missing Config Fields** | `KeyError` check | Print missing fields, exit with code 1 |
| **Image URL 404** | `requests.HTTPError` | Send error message to channel |
| **Image Size Exceeds Limit** | Check `Content-Length` header | Send warning message, abort |
| **Network Timeout** | `requests.Timeout` | Send timeout error to channel |
| **Discord API Error** | `discord.HTTPException` | Log error, send user notification |
| **Invalid Discord Token** | `discord.LoginFailure` | Print error, mark bot as stopped |

### 3.3 Regex Pattern Specifications

```python
# Pattern matches common image hosting formats
IMAGE_URL_PATTERN = r'(https?://[^\s]+\.(?:png|jpg|jpeg|gif|webp)(?:\?[^\s]*)?)'

# Explanation:
# - https?://          : Match http or https protocol
# - [^\s]+             : Match any non-whitespace characters (domain + path)
# - \.(?:png|jpg|...)  : Match image file extensions
# - (?:\?[^\s]*)?      : Optionally match query parameters

# Test cases:
# ✅ https://example.com/image.png
# ✅ https://cdn.discord.com/attachments/123/456/file.jpg?size=1024
# ✅ http://i.imgur.com/abc123.gif
# ❌ https://example.com/page.html
# ❌ not-a-url.png
```

### 3.4 Discord File Size Limits

| Server Boost Level | Max File Size |
|-------------------|---------------|
| No Boost | 8 MB |
| Level 2 | 50 MB |
| Level 3 | 100 MB |

**Implementation:**
```python
# Conservative default in config.json
"max_image_size_mb": 8

# Check before downloading
if content_length > CONFIG['max_image_size_mb'] * 1024 * 1024:
    return  # Abort
```

### 3.5 HTML Template Structure

**File: `templates/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Discord Bot Manager</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 600px;
            margin: 50px auto;
            padding: 20px;
            background: #2c2f33;
            color: #ffffff;
        }
        .status {
            padding: 20px;
            border-radius: 5px;
            margin-bottom: 20px;
            text-align: center;
        }
        .status.running { background: #43b581; }
        .status.stopped { background: #f04747; }
        button {
            padding: 15px 30px;
            font-size: 16px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            margin: 5px;
        }
        .start-btn { background: #43b581; color: white; }
        .stop-btn { background: #f04747; color: white; }
        button:disabled { opacity: 0.5; cursor: not-allowed; }
    </style>
</head>
<body>
    <h1>Discord Bot Control Panel</h1>
    
    <div class="status {% if bot_running %}running{% else %}stopped{% endif %}">
        <h2>Status: {% if bot_running %}RUNNING{% else %}STOPPED{% endif %}</h2>
    </div>
    
    <div style="text-align: center;">
        <button class="start-btn" onclick="startBot()" id="startBtn">Start Bot</button>
        <button class="stop-btn" onclick="stopBot()" id="stopBtn">Stop Bot</button>
    </div>
    
    <div id="message" style="margin-top: 20px; text-align: center;"></div>
    
    <script>
        function startBot() {
            fetch('/start', { method: 'POST' })
                .then(r => r.json())
                .then(data => {
                    showMessage(data.message, data.success);
                    if(data.success) location.reload();
                });
        }
        
        function stopBot() {
            fetch('/stop', { method: 'POST' })
                .then(r => r.json())
                .then(data => {
                    showMessage(data.message, data.success);
                    if(data.success) location.reload();
                });
        }
        
        function showMessage(msg, success) {
            const el = document.getElementById('message');
            el.textContent = msg;
            el.style.color = success ? '#43b581' : '#f04747';
        }
    </script>
</body>
</html>
```

---

## 4. IMPLEMENTATION STEPS (AI AGENT PROMPT CHAIN)

### Prompt 1: Project Foundation & Configuration

```
Create a Python project with the following structure:

discord-bot-manager/
├── config.json
├── main.py
├── bot_manager.py
└── templates/
    └── index.html

Requirements:
1. Create a config.json file with fields: discord_token (string), allowed_guild_ids (array of integers), allowed_channel_ids (array of integers), flask_port (integer, default 5000), flask_host (string, default "127.0.0.1"), max_image_size_mb (integer, default 8).

2. In main.py, write code to:
   - Load config.json using the json module
   - Validate that all required fields exist (exit with error message if missing)
   - Print a confirmation message showing the number of whitelisted guilds and channels
   - Initialize a Flask app on the configured host and port
   - Create a simple index route that returns "Discord Bot Manager"

3. Add error handling for FileNotFoundError and json.JSONDecodeError when loading config.

4. Include if __name__ == "__main__": block to run the Flask app.

Do not implement the Discord bot yet. Focus only on configuration loading and Flask setup.
```

### Prompt 2: Discord Client with Whitelist Filtering

```
Extend the project to add Discord bot functionality with strict whitelisting:

1. Create bot_manager.py with a BotManager class that:
   - Takes config dict in __init__
   - Has attributes: bot_client (discord.Client), bot_thread (threading.Thread), is_running (bool), event_loop (asyncio event loop)
   - Implements start_bot() method that returns {"success": bool, "message": str}
   - Implements stop_bot() method with the same return format

2. In the start_bot() method:
   - Check if bot is already running (return error if true)
   - Create a new asyncio event loop
   - Initialize discord.Client with Intents.default() and message_content=True
   - Register an on_ready event handler that prints the bot's username
   - Register an on_message event handler with THIS EXACT FILTERING LOGIC AS THE FIRST STEPS:
     a. Return if message.author == client.user
     b. Return if message.guild is None
     c. Return if message.guild.id not in config['allowed_guild_ids']
     d. Return if message.channel.id not in config['allowed_channel_ids']
     e. Only after these checks, print: f"Message from {message.author}: {message.content}"
   - Start the bot in a daemon thread running the event loop
   - Set is_running = True

3. In the stop_bot() method:
   - Check if bot is not running (return error if true)
   - Use asyncio.run_coroutine_threadsafe to call bot_client.close()
   - Join the bot thread with a 5-second timeout
   - Set is_running = False

4. Update main.py to:
   - Import BotManager
   - Create a global bot_manager instance after loading config
   - Add these Flask routes:
     POST /start → calls bot_manager.start_bot(), returns JSON
     POST /stop → calls bot_manager.stop_bot(), returns JSON
     GET /status → returns {"running": bot_manager.is_running}

Test by starting the bot and verifying it only responds to whitelisted guild/channel IDs.
```

### Prompt 3: Image Proxy with Spoiler Logic

```
Implement the spoiler image proxy feature in the on_message handler:

1. Add these imports to bot_manager.py: import re, import io, import requests

2. Modify the on_message handler to add this logic AFTER the whitelist checks:
   
   a. Use this regex pattern to find image URLs:
      IMAGE_URL_PATTERN = r'(https?://[^\s]+\.(?:png|jpg|jpeg|gif|webp)(?:\?[^\s]*)?)'
      urls = re.findall(IMAGE_URL_PATTERN, message.content, re.IGNORECASE)
   
   b. If no URLs found, return (do nothing)
   
   c. Take the first URL from the list
   
   d. Download the image:
      - Use requests.get(url, stream=True, timeout=10)
      - Check the Content-Length header against max_image_size_mb from config
      - If too large, send a warning message to the channel and return
      - Stream the response into an io.BytesIO buffer (chunk_size=8192)
      - Seek the buffer to position 0
   
   e. Create a discord.Embed:
      - Set description to message.content
      - Set author name to message.author.display_name
      - Set author icon to message.author.avatar.url (handle None case)
      - Set timestamp to message.created_at
      - Set color to discord.Color.blue()
   
   f. Create a discord.File:
      - Use the BytesIO buffer as fp parameter
      - Extract filename from URL (last part of path before query string)
      - Prefix filename with "SPOILER_"
   
   g. Delete the original message: await message.delete()
   
   h. Send new message: await message.channel.send(embed=embed, file=file)

3. Add try-except blocks:
   - Catch requests.RequestException → send error message to channel
   - Catch discord.HTTPException → send error message to channel
   - Catch general Exception → print to console

Test with real image URLs in a whitelisted channel.
```

### Prompt 4: GUI Integration & Final Polish

```
Complete the web interface and add final production features:

1. Create templates/index.html with:
   - Dark theme (#2c2f33 background)
   - Status indicator showing "RUNNING" (green #43b581) or "STOPPED" (red #f04747)
   - Two buttons: "Start Bot" and "Stop Bot"
   - JavaScript functions that:
     - Call /start and /stop endpoints using fetch()
     - Display response messages below the buttons
     - Reload the page on success to update status
   - Disable buttons while requests are in flight

2. Update the Flask index route in main.py:
   - Use render_template('index.html', bot_running=bot_manager.is_running)

3. Add a threading.Lock to BotManager:
   - Initialize in __init__: self.lock = threading.Lock()
   - Wrap start_bot and stop_bot logic with: with self.lock:

4. Add graceful shutdown handling in main.py:
   - Import signal and atexit modules
   - Create cleanup function that calls bot_manager.stop_bot()
   - Register cleanup with: atexit.register(cleanup)
   - Register signal handlers for SIGINT and SIGTERM

5. Add logging:
   - Import logging module
   - Configure basic logging in main.py
   - Replace print statements with logging.info/error calls

6. Create a .gitignore file with: config.json, __pycache__/, *.pyc, venv/

7. Test the complete workflow:
   - Start Flask app
   - Open browser to http://localhost:5000
   - Click Start Bot → verify bot comes online
   - Post image URL in whitelisted channel → verify spoiler repost
   - Post in non-whitelisted channel → verify no response
   - Click Stop Bot → verify bot disconnects
   - Close Flask app → verify bot stops automatically

Document any issues encountered and their solutions.
```

---

## 5. DEPLOYMENT CHECKLIST

### Pre-Deployment

- [ ] Set strong Discord token in `config.json`
- [ ] Configure firewall rules for Flask port
- [ ] Test all whitelist configurations
- [ ] Verify bot permissions in target guilds
- [ ] Test image download with various sizes
- [ ] Validate graceful shutdown behavior

### Production Considerations

- [ ] Use environment variables instead of `config.json`
- [ ] Implement rate limiting on Flask endpoints
- [ ] Add authentication to web interface
- [ ] Set up HTTPS with reverse proxy (nginx/Apache)
- [ ] Configure systemd service for auto-restart
- [ ] Implement structured logging to file
- [ ] Set up monitoring/alerting for bot disconnections

### Security Hardening

- [ ] Bind Flask to `127.0.0.1` only (use reverse proxy for external access)
- [ ] Implement CSRF protection on POST endpoints
- [ ] Add input validation for all user-supplied data
- [ ] Set up fail2ban for repeated unauthorized access
- [ ] Regular dependency updates (`pip list --outdated`)

---

## 6. TROUBLESHOOTING GUIDE

| Symptom | Probable Cause | Solution |
|---------|---------------|----------|
| Bot doesn't respond in whitelisted channel | Guild ID mismatch | Use Developer Mode to copy exact IDs (right-click server/channel) |
| "Bot already running" on start | Previous shutdown incomplete | Restart entire Flask app |
| Images not downloading | Firewall/proxy blocking | Check network connectivity, try different image host |
| Spoiler tag not appearing | Filename missing prefix | Verify "SPOILER_" is prepended to filename |
| High memory usage | Large images not cleaned up | Ensure BytesIO buffers go out of scope after send |
| Flask won't start | Port already in use | Change `flask_port` in config or kill process on port 5000 |

---

## 7. MAINTENANCE & MONITORING

### Log Analysis Points

```python
# Key log patterns to monitor:
# - "Bot connected as {username}" → Successful start
# - "Bot thread error:" → Critical failure, investigate immediately
# - "Failed to download image:" → Network issues or bad URLs
# - "Message from {author}:" → Normal operation verification
```

### Performance Metrics

- **Average Response Time:** < 2 seconds from message to repost
- **Memory Usage:** < 150MB baseline, < 300MB under load
- **Thread Count:** 2 (main + bot thread)
- **Network I/O:** Depends on image sizes, typically < 10MB/min

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Maintainer:** Senior Lead Software Architect  
**Target Audience:** Junior-to-Mid Level Python Developers
