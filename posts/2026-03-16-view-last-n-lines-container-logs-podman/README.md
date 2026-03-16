# How to View Last N Lines of Container Logs in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Logging

Description: Learn how to use the --tail flag in Podman to view only the last N lines of container logs, making it easier to focus on recent output without scrolling through the entire log history.

---

> When a container has been running for days, you rarely need to see all its logs. The --tail flag lets you jump straight to the most recent output.

Long-running containers can accumulate thousands or millions of log lines. Viewing the entire log is slow and overwhelming. The `--tail` flag in `podman logs` lets you view only the most recent lines, giving you a quick snapshot of current container behavior.

---

## Basic Usage of --tail

```bash
# View the last 10 lines of container logs
podman logs --tail 10 my-container

# View the last 100 lines
podman logs --tail 100 my-container

# View only the very last line
podman logs --tail 1 my-container

# View the last 50 lines with timestamps
podman logs --tail 50 --timestamps my-container
```

## Combine --tail with --follow

Start following logs from a specific number of recent lines.

```bash
# Show the last 20 lines and then follow new output
podman logs --tail 20 -f my-container

# Follow with no history (only new lines)
podman logs --tail 0 -f my-container

# Show last 5 lines with timestamps, then follow
podman logs --tail 5 --timestamps -f my-container
```

## View Last N Lines from Stopped Containers

The `--tail` flag works on stopped containers too, which is useful for post-mortem debugging.

```bash
# List stopped containers
podman ps -a --filter status=exited

# View the last 50 lines from a stopped container
podman logs --tail 50 stopped-container

# View the last 20 lines from the most recently created container
podman logs --tail 20 $(podman ps -lq)

# View last lines from all stopped containers
for c in $(podman ps -a --filter status=exited --format '{{.Names}}'); do
  echo "=== Last 10 lines from $c ==="
  podman logs --tail 10 "$c"
  echo ""
done
```

## Tail Logs with Filtering

Combine `--tail` with text processing tools to find specific information.

```bash
# Get the last 200 lines and search for errors
podman logs --tail 200 my-container 2>&1 | grep -i error

# Get the last 500 lines and count errors vs warnings
podman logs --tail 500 my-container 2>&1 | grep -c -i error
podman logs --tail 500 my-container 2>&1 | grep -c -i warn

# Get the last 100 lines and extract unique error messages
podman logs --tail 100 my-container 2>&1 | grep -i error | sort -u

# View last N lines of only stderr
podman logs --tail 50 my-container 2>&1 1>/dev/null
```

## Tail Logs from Multiple Containers

Quickly check recent output from several containers.

```bash
# View last 5 lines from each running container
for c in $(podman ps --format '{{.Names}}'); do
  echo "=== $c ==="
  podman logs --tail 5 "$c" 2>&1
  echo ""
done

# View last 10 lines from containers matching a pattern
for c in $(podman ps --format '{{.Names}}' | grep web); do
  echo "--- $c ---"
  podman logs --tail 10 "$c" 2>&1
  echo ""
done

# Quick health check: show last line from all containers
podman ps --format '{{.Names}}' | while read -r c; do
  LAST_LINE=$(podman logs --tail 1 "$c" 2>&1)
  echo "$c: $LAST_LINE"
done
```

## Practical Examples

```bash
# Check if a web server started successfully
podman logs --tail 5 nginx-container
# Expected output: lines showing nginx is listening

# Check the last few database queries
podman logs --tail 20 postgres-container 2>&1 | grep "LOG:"

# Check recent authentication failures
podman logs --tail 100 auth-service 2>&1 | grep -i "auth.*fail"

# Monitor recent deployment output
podman logs --tail 30 --timestamps deploy-container
```

## Use in Shell Scripts

Automate log tail checks in monitoring scripts.

```bash
#!/bin/bash
# check-containers.sh - Quick health check via recent logs

CONTAINERS=("web" "api" "worker" "db")

for c in "${CONTAINERS[@]}"; do
  # Check if container is running
  if podman ps --format '{{.Names}}' | grep -q "^${c}$"; then
    # Count errors in last 100 lines
    ERROR_COUNT=$(podman logs --tail 100 "$c" 2>&1 | grep -ci error)
    if [ "$ERROR_COUNT" -gt 0 ]; then
      echo "WARNING: $c has $ERROR_COUNT errors in last 100 log lines"
      podman logs --tail 5 "$c" 2>&1 | grep -i error
    else
      echo "OK: $c - no recent errors"
    fi
  else
    echo "DOWN: $c is not running"
  fi
done
```

## Summary

The `--tail N` flag is essential for working with long-running containers. It limits output to the most recent N lines, reducing noise and improving performance. Combine it with `--follow` to start streaming from a recent point, or with `grep` to quickly search recent output. Use `--tail 0 -f` when you only care about new log lines going forward.
