# MCP Agent Mail Quick Reference

## Server Status

**Service:** `com.groovejones.mcp-agent-mail`
**URL:** http://127.0.0.1:8765/mcp/
**Web UI:** http://127.0.0.1:8765/mail
**Logs:**

- stdout: `~/Library/Logs/mcp-agent-mail.log`
- stderr: `~/Library/Logs/mcp-agent-mail-error.log`

## Service Control

```bash
# Check status
launchctl list | grep mcp-agent-mail

# View logs
tail -f ~/Library/Logs/mcp-agent-mail.log

# Stop service
launchctl stop com.groovejones.mcp-agent-mail

# Start service
launchctl start com.groovejones.mcp-agent-mail

# Restart service
launchctl kickstart -k gui/$(id -u)/com.groovejones.mcp-agent-mail

# Disable service (unload)
launchctl unload ~/Library/LaunchAgents/com.groovejones.mcp-agent-mail.plist

# Enable service (load)
launchctl load ~/Library/LaunchAgents/com.groovejones.mcp-agent-mail.plist
```

## Product Bus: GrooveTech-Orchestrator-Suite

**Linked Repos:**

1. groovetech-media-player (GMP)
2. orchestrator
3. AVPStreamKit
4. PfizerOutdoCancerV2 (Pfizer)
5. groovetech-media-server (Media Server)

```bash
# Product status
uv run python -m mcp_agent_mail.cli products status GrooveTech-Orchestrator-Suite

# Search across all repos
uv run python -m mcp_agent_mail.cli products search GrooveTech-Orchestrator-Suite "bd-123"

# Agent inbox (product-wide)
uv run python -m mcp_agent_mail.cli products inbox GrooveTech-Orchestrator-Suite YourAgentName --limit 20

# Summarize thread across repos
uv run python -m mcp_agent_mail.cli products summarize-thread GrooveTech-Orchestrator-Suite "bd-123"
```

## Agent Workflow

### 1. Session Start

```python
agentmail_init(
  project_path="/abs/path/to/repo",
  task_description="Working on feature X"
)
# Returns: { agent_name: "BlueLake", project_key: "..." }
```

### 2. Before Editing Files

```python
agentmail_reserve(
  patterns=["Sources/**"],
  ttl_seconds=3600,
  exclusive=True
)
```

### 3. Communicate

```python
# Send message
agentmail_send(
  to="OtherAgent",
  subject="API changes in AVPStreamKit",
  body="Updated the streaming protocol...",
  thread_id="bd-123"
)

# Check inbox (headers only, limit 5)
agentmail_inbox()

# Read specific message
agentmail_read_message(message_id="...")

# Acknowledge important message
agentmail_ack(message_id="...")
```

### 4. After Work

```python
agentmail_release()  # Release all reservations
```

## Integration with Beads

- Use beads issue ID as `thread_id` (e.g., `thread_id="bd-123"`)
- Include issue ID in file reservation `reason`
- When starting a beads task, reserve files; when closing, release them

```bash
# Pick ready work
bd ready --json | jq '.[0]'

# Start task + reserve files
bd update bd-123 --status in_progress
agentmail_reserve(patterns=["Sources/**"], reason="bd-123")

# Complete task + release
bd close bd-123 --reason "Done"
agentmail_release()
```

## Troubleshooting

### Server not responding

```bash
# Check if running
curl http://127.0.0.1:8765/health/liveness

# Check logs
tail -50 ~/Library/Logs/mcp-agent-mail-error.log

# Restart
launchctl kickstart -k gui/$(id -u)/com.groovejones.mcp-agent-mail
```

### Port conflict (8765 in use)

```bash
# Find process using port
lsof -i :8765

# Change port in .env (in mcp_agent_mail repo)
echo "HTTP_PORT=9000" >> .env

# Restart service
launchctl kickstart -k gui/$(id -u)/com.groovejones.mcp-agent-mail
```

### Agent not registered error

```python
# Always init first
agentmail_init(project_path="...", task_description="...")
```

### File reservation conflict

```python
# Check who has the reservation
agentmail_search(query="file_reservation Sources/**")

# Wait for expiry or use shared reservation
agentmail_reserve(patterns=["Sources/**"], exclusive=False)
```

## Web UI Routes

- **Dashboard:** http://127.0.0.1:8765/mail
- **Projects:** http://127.0.0.1:8765/mail/projects
- **Project detail:** http://127.0.0.1:8765/mail/{project_slug}
- **Agent inbox:** http://127.0.0.1:8765/mail/{project_slug}/inbox/{agent_name}
- **Message:** http://127.0.0.1:8765/mail/{project_slug}/message/{message_id}
- **Search:** http://127.0.0.1:8765/mail/{project_slug}/search?q=...
- **File reservations:** http://127.0.0.1:8765/mail/{project_slug}/file_reservations
