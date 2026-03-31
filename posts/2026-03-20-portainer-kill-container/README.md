# How to Kill a Running Container in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Container, Operation, DevOps

Description: Learn how to forcefully terminate a running Docker container in Portainer using the kill command, and when to use it versus a graceful stop.

## Introduction

While `docker stop` sends a graceful SIGTERM signal to allow a container to shut down cleanly, `docker kill` sends an immediate SIGKILL (or a custom signal) that terminates the container without waiting. Portainer provides this capability for situations where a container is unresponsive or needs immediate termination.

## Prerequisites

- Portainer installed with a connected Docker environment
- A running container to terminate

## Stop vs. Kill

Understanding the difference:

```bash
docker stop my-container
  1. Sends SIGTERM (graceful shutdown signal)
  2. Waits up to --time seconds (default: 10)
  3. If still running: sends SIGKILL

docker kill my-container
  1. Immediately sends SIGKILL (no grace period)
  2. Container is terminated instantly
  (Or sends any specified signal)
```

Kill is appropriate when:
- The container is unresponsive to stop
- You need immediate termination
- The container is in a bad state and graceful shutdown doesn't matter
- You need to send a specific signal (e.g., SIGHUP for config reload)

## Step 1: Kill a Container in Portainer

### Via the Container List

1. Navigate to **Containers** in Portainer.
2. Find the running container.
3. Click the **Kill** button (X or lightning icon - may vary by Portainer version).

Note: In some Portainer versions, Kill may only be available from the container details page.

### Via the Container Details Page

1. Click on the container name.
2. Click the **Kill** button in the action bar.

```bash
# Equivalent Docker CLI:

docker kill my-container

# Send a specific signal:
docker kill --signal SIGHUP my-container   # Config reload
docker kill --signal SIGUSR1 my-container  # Custom handler
docker kill --signal SIGTERM my-container  # Same as stop
```

## Step 2: Common Signals and Their Uses

| Signal | Value | Meaning | Common Use |
|--------|-------|---------|-----------|
| `SIGTERM` | 15 | Graceful termination | Normal shutdown |
| `SIGKILL` | 9 | Immediate kill | Force-terminate unresponsive container |
| `SIGHUP` | 1 | Hangup / reload | Reload config (nginx, sshd) |
| `SIGINT` | 2 | Interrupt | Like Ctrl+C |
| `SIGUSR1` | 10 | User-defined | Application-specific |
| `SIGUSR2` | 12 | User-defined | Application-specific (e.g., log rotate) |

## Step 3: Killing Unresponsive Containers

When a container doesn't respond to stop:

```bash
# Attempt graceful stop with short timeout:
docker stop --time 5 my-hung-container

# If that fails, force kill:
docker kill my-hung-container

# In Portainer:
# 1. Click Stop (waits grace period)
# 2. If container is still "Stopping" after a minute, click Kill
```

## Step 4: Using Kill to Reload Configuration

Some services reload config on SIGHUP without restarting:

```bash
# Reload nginx configuration without downtime:
docker kill --signal SIGHUP my-nginx

# Reload HAProxy config:
docker kill --signal SIGUSR2 my-haproxy
```

For custom applications handling specific signals:

```python
# Python app handling SIGUSR1 for graceful log rotation
import signal
import logging

def handle_log_rotate(signum, frame):
    logging.info("Received SIGUSR1 - rotating logs")
    # Re-open log files
    for handler in logging.root.handlers[:]:
        handler.close()
        logging.root.removeHandler(handler)
    logging.basicConfig(filename='/app/logs/app.log')

signal.signal(signal.SIGUSR1, handle_log_rotate)
```

## Step 5: After Killing a Container

After killing, the container enters the **Stopped** (Exited) state:

1. In Portainer, the container shows as stopped.
2. You can view its last logs before the kill.
3. You can restart it or remove it.

Check the exit status:

```bash
docker inspect my-container | jq '.[].State'

# Output for kill:
{
  "Status": "exited",
  "ExitCode": 137,    # 128 + 9 (SIGKILL) = 137
  "OOMKilled": false
}
```

Exit code 137 = killed by SIGKILL (either manually or by OOM killer).

## Preventing Situations That Require Kill

Design your applications to respond to SIGTERM cleanly:

```bash
#!/bin/sh
# Entrypoint with proper signal handling

# Forward SIGTERM to the application process
_term() {
  echo "Received SIGTERM - shutting down gracefully"
  kill -TERM "$child" 2>/dev/null
}

trap _term SIGTERM

# Start the app in the background
./my-app &
child=$!

# Wait for the app to finish
wait "$child"
exit $?
```

In docker-compose.yml, use init to ensure proper signal forwarding:

```yaml
services:
  app:
    image: myorg/myapp:latest
    init: true   # Use Docker's tini init for proper signal handling
    stop_grace_period: 30s
    stop_signal: SIGTERM
```

## When NOT to Use Kill

- **Data integrity**: Killing a database container mid-write may corrupt data.
- **Ongoing transactions**: Kill drops transactions without rollback.
- **File write operations**: Files may be left in a partially written state.

Prefer stop (with adequate grace period) for services with persistent state.

## Conclusion

The kill command in Portainer provides immediate container termination when graceful shutdown isn't possible or practical. Use it for unresponsive containers, signal-based config reloads, and situations requiring immediate termination. For regular operations, always prefer stop to allow applications to clean up gracefully. Design your applications to handle SIGTERM properly to minimize the need for force kills.
