# How to Enable Debug Logging for Troubleshooting in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Debugging, Logging, Troubleshooting, CLI Flags, Docker

Description: Learn how to enable and use debug logging in Portainer to diagnose connection issues, authentication failures, and API errors during troubleshooting.

---

Debug logging in Portainer produces verbose output including API calls, internal state transitions, and detailed error messages. It is invaluable for diagnosing issues that INFO-level logs don't explain.

## Enabling Debug Logging

Add `--log-level DEBUG` to the Portainer startup command:

```bash
# Stop and remove the current Portainer container

docker stop portainer && docker rm portainer

# Start with debug logging
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level DEBUG
```

## Capturing Debug Logs

```bash
# Follow debug logs in real-time
docker logs -f portainer 2>&1

# Save to a file for analysis
docker logs portainer 2>&1 > portainer-debug.log

# Filter for specific issues
docker logs portainer 2>&1 | grep -i "error\|failed\|timeout"

# Watch for agent-specific debug messages
docker logs -f portainer 2>&1 | grep -i "agent\|tunnel\|edge"
```

## What Debug Logs Show

Debug mode reveals:

```bash
# Example debug log entries:
# HTTP API request details
level=debug msg="HTTP request" endpoint="/api/containers/json" method=GET status=200 duration=42ms

# Docker API calls
level=debug msg="Calling Docker API" url="/v1.41/containers/json" endpoint=1

# Authentication events
level=debug msg="JWT token validated" user=admin userId=1

# Snapshot operations
level=debug msg="Creating environment snapshot" environmentId=1 name=local

# Agent communication
level=debug msg="Sending data to agent" agentAddress=192.168.1.100:9001
```

## Temporary Debug Session

For production systems, enable debug logging temporarily, capture what you need, then restore normal logging:

```bash
# 1. Enable debug logging
docker stop portainer && docker rm portainer
docker run -d ... portainer/portainer-ce:latest --log-level DEBUG

# 2. Reproduce the issue and capture logs
docker logs portainer 2>&1 | tail -200 > debug-capture.log

# 3. Restore normal logging
docker stop portainer && docker rm portainer
docker run -d ... portainer/portainer-ce:latest  # No --log-level flag (defaults to INFO)
```

## JSON Debug Logs

Combine debug level with JSON format for structured analysis:

```bash
docker run -d ... portainer/portainer-ce:latest \
  --log-level DEBUG \
  --log-mode JSON

# Parse with jq to find slow operations
docker logs portainer 2>&1 | \
  jq -r 'select(.level == "debug" and .duration != null) | "\(.duration) \(.msg)"' | \
  sort -n | tail -20
```
