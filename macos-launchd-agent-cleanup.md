
# macOS Launchd Agent Cleanup

## Complete Removal Workflow

When you need to fully remove a launchd agent (e.g., a service like openclaw, a background daemon):

### Step 1: Identify the agent and kill its process
```bash
# Find what's using the port
lsof -i :<PORT>

# Find running processes
ps aux | grep -i <NAME>

# Kill by PID
kill <PID>
```

### Step 2: Unload the agent (stops it but may leave registration)
```bash
launchctl unload ~/Library/LaunchAgents/<name>.plist
```

### Step 3: Bootout (THE KEY STEP — fully removes from launchd domain)
```bash
launchctl bootout gui/$(id -u)/<label>
# Example: launchctl bootout gui/501/ai.openclaw.gateway
```

### Step 4: Delete plist file
```bash
rm -f ~/Library/LaunchAgents/<name>.plist
```

### Step 5: Verify
```bash
launchctl list | grep -i <NAME>
lsof -i :<PORT>  # port should be free
ps aux | grep -i <NAME>  # no processes
```

## Common Pitfalls

### `launchctl unload` alone is NOT enough
- It stops the job but does NOT remove it from the launchd namespace
- `launchctl list` will STILL show the agent with a non-zero exit code
- You MUST use `bootout` to fully deregister

### `launchctl remove` is also insufficient for user agents
- `remove` works for system-wide agents
- For user agents (gui/UID/), use `bootout gui/UID/<label>`

### Order matters
1. Kill the process first
2. Then bootout (this also stops the job)
3. Then delete plist (safe to delete after bootout)

## Finding agents

```bash
# User agents
ls ~/Library/LaunchAgents/

# System-wide agents (requires sudo for inspection)
ls /Library/LaunchAgents/

# All running agents matching name
launchctl list | grep -i <NAME>

# By port
lsof -i :<PORT>
```
