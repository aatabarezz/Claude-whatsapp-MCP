# WhatsApp MCP Project Configuration

**Status**: ✅ Active & Running  
**Last Updated**: 2026-09-11  
**Owner**: altanatabarezz@gmail.com

## Quick Reference

### Launch Commands

```bash
# Check status
launchctl list | grep whatsapp

# View real-time logs
tail -f /tmp/whatsapp-mcp.log

# Manual start (without LaunchAgent)
cd ~/whatsapp-mcp/whatsapp-bridge && go run main.go

# Restart service
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.whatsapp-mcp.bridge.plist
sleep 2
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.whatsapp-mcp.bridge.plist
```

## Project Structure

```
~/whatsapp-mcp/
├── whatsapp-bridge/              # Go bridge component
│   ├── main.go                   # Entry point
│   ├── go.mod / go.sum           # Go dependencies
│   └── store/                    # SQLite databases
│       ├── whatsapp.db           # Session state
│       └── messages.db           # Message history
│
├── whatsapp-mcp-server/          # Python MCP server
│   ├── main.py                   # MCP server implementation
│   ├── pyproject.toml            # Python dependencies
│   └── .venv/                    # Virtual environment
│
├── bin/
│   └── start-bridge.sh           # LaunchAgent startup script
│
├── README.md                     # Original documentation
└── PROJECT.md                    # This file
```

## Setup Details

### Installation Date
2026-06-22 (initial setup)  
2026-09-11 (persistent LaunchAgent configuration)

### Component Versions
- **Go**: 1.26.4 (installed via Homebrew at `/opt/homebrew/bin/go`)
- **Python**: 3.12.2
- **UV**: 0.11.21 (installed at `~/.local/bin/uv`)
- **FFmpeg**: 8.1.1 (installed via Homebrew)
- **whatsmeow**: Latest from go.mod (as of 2026-06-22)

### Configuration Files

**Claude Desktop MCP Config**  
Location: `~/Library/Application Support/Claude/claude_desktop_config.json`
```json
{
  "mcpServers": {
    "whatsapp": {
      "command": "~/.local/bin/uv",
      "args": [
        "--directory",
        "~/whatsapp-mcp/whatsapp-mcp-server",
        "run",
        "main.py"
      ]
    }
  }
}
```

**LaunchAgent Configuration**  
Location: `~/Library/LaunchAgents/com.whatsapp-mcp.bridge.plist`  
- Runs at user login
- Auto-restarts on crash
- Logs to `/tmp/whatsapp-mcp.log` and `/tmp/whatsapp-mcp-error.log`

## Architecture

```
WhatsApp Web
    ↓
Go Bridge (whatsmeow library)
    ↓
SQLite Database (local)
    ↓
Python MCP Server
    ↓
Claude Desktop
```

### Data Flow
1. Go bridge connects to WhatsApp Web multidevice API
2. Messages synced continuously to local SQLite
3. Python MCP server reads from SQLite
4. Claude requests through MCP tools trigger Python server
5. Results returned to Claude

### Security Notes
- ✅ All data stored locally in SQLite
- ✅ Messages only sent to Claude on explicit request
- ✅ No cloud backup or sync
- ✅ QR code authentication via WhatsApp mobile app
- ⚠️ Subject to "lethal trifecta" risk (prompt injection → data exfiltration)

## Available Tools in Claude

12 MCP tools exposed:
- `list_chats` — Get chat list with metadata
- `search_contacts` — Find contacts by name/phone
- `list_messages` — Read messages from chat
- `get_chat` — Get chat info
- `get_direct_chat_by_contact` — Find DM with contact
- `get_contact_chats` — List all chats with contact
- `get_last_interaction` — Get most recent message
- `get_message_context` — Get messages around a specific message
- `send_message` — Send text to individual/group
- `send_file` — Send image/video/document/audio
- `send_audio_message` — Send voice message
- `download_media` — Download media from message

## Operational Status

### Last Known Status (2026-09-11)
- ✅ Go bridge running with PID 92524
- ✅ REST API active on :8080
- ✅ Messages syncing in real-time
- ✅ Claude Desktop connected
- ✅ All tools operational

### Performance Characteristics
- **Initial sync**: 2-5 minutes for first run
- **Message retrieval**: <100ms per query
- **Media download**: 1-10s depending on file size
- **Send latency**: 1-3s per message
- **Concurrent chats**: Supports 1000s of chats
- **Re-authentication**: Required every ~20 days

## Troubleshooting Guide

### Issue: Bridge won't start

**Check 1: Go installation**
```bash
which go  # Should return /opt/homebrew/bin/go
go version
```

**Check 2: Port availability**
```bash
lsof -i :8080  # Should be empty
```

**Check 3: Dependencies**
```bash
cd ~/whatsapp-mcp/whatsapp-bridge
go mod tidy && go run main.go
```

### Issue: Messages not syncing

1. Wait 2-5 minutes (initial sync)
2. Check logs: `tail -50 /tmp/whatsapp-mcp.log`
3. Restart: Delete databases + re-authenticate
   ```bash
   rm ~/whatsapp-mcp/whatsapp-bridge/store/*.db
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.whatsapp-mcp.bridge.plist
   sleep 2
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.whatsapp-mcp.bridge.plist
   ```

### Issue: Claude can't find WhatsApp

1. Verify config JSON syntax: `plutil -lint ~/Library/Application\ Support/Claude/claude_desktop_config.json`
2. Check Python server: `ps aux | grep python3 | grep main.py`
3. Restart Claude Desktop completely
4. Verify paths match actual installation

### Issue: QR code not displaying

- Try different terminal (iTerm2, etc.)
- Restart: `go run main.go`
- Check for terminal Unicode support

### Issue: Media files won't download

- Verify disk space: `df -h`
- Check permissions: `ls -la /tmp/`
- Ensure file isn't corrupted in SQLite

## Maintenance

### Regular Tasks
- **Weekly**: Check logs for errors
- **Monthly**: Verify service is still running
- **Every 20 days**: May need to re-authenticate (new QR code)

### Updating Dependencies
```bash
# Update Go
brew upgrade go

# Update Python dependencies
cd ~/whatsapp-mcp/whatsapp-mcp-server
uv sync

# Update whatsmeow
cd ~/whatsapp-mcp/whatsapp-bridge
go get -u go.mau.fi/whatsmeow
```

## References & Links

- **Repository**: https://github.com/lharries/whatsapp-mcp
- **whatsmeow**: https://github.com/tulir/whatsmeow
- **MCP Spec**: https://modelcontextprotocol.io/
- **Claude Desktop Docs**: https://claude.ai/download
- **LaunchAgent Docs**: https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchAgents.html

## Notes

- This is a personal integration for altanatabarezz@gmail.com
- All data is private and local
- Bridge requires active internet connection
- Works best with modern macOS (10.14+)
- WhatsApp limits concurrent device sessions

## Future Improvements

Potential enhancements:
- [ ] Scheduled message sending
- [ ] Message search with NLP
- [ ] Group management tools
- [ ] Status update reading
- [ ] Call log integration
- [ ] Database backup automation
- [ ] Performance monitoring
- [ ] Webhook notifications
