# How to Stop a Running Container in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Container Lifecycle, Container Management

Description: Learn how to gracefully stop running containers in Podman with configurable timeouts and signal handling.

---

> Stopping containers gracefully ensures that applications can clean up resources and save state before shutting down.

Stopping a container in Podman sends a SIGTERM signal to the main process, giving it time to shut down cleanly. If the process does not exit within the timeout period, Podman sends a SIGKILL to force termination. This guide covers the different ways to stop containers and configure shutdown behavior.

---

## Basic Container Stop

Stop a running container by name or ID.

```bash
# Start a container

podman run -d --name my-nginx docker.io/library/nginx:latest

# Stop the container gracefully
podman stop my-nginx

# Verify it stopped
podman ps -a --filter name=my-nginx --format "{{.Names}} {{.Status}}"
```

## Setting a Custom Timeout

By default, Podman waits 10 seconds for the container to stop before sending SIGKILL. Customize this with the `--time` (or `-t`) flag.

```bash
# Stop with a 30-second grace period
podman stop --time 30 my-nginx

# Stop with a 5-second grace period
podman stop -t 5 my-nginx

# Stop immediately with no grace period (sends SIGKILL right away)
podman stop -t 0 my-nginx
```

## Stopping by Container ID

```bash
# Get the container ID
CONTAINER_ID=$(podman ps -q --filter name=my-nginx)

# Stop using the ID
podman stop "$CONTAINER_ID"
```

## Stopping Multiple Containers

Stop several containers in one command.

```bash
# Start multiple containers
podman run -d --name web1 docker.io/library/nginx:latest
podman run -d --name web2 docker.io/library/nginx:latest
podman run -d --name web3 docker.io/library/nginx:latest

# Stop all three at once
podman stop web1 web2 web3

# Verify
podman ps
```

## Stopping All Running Containers

```bash
# Stop all running containers
podman stop --all

# Alternative: pipe container IDs to podman stop
podman ps -q | xargs -r podman stop
```

## Understanding the Stop Process

When you run `podman stop`, this is what happens:

1. Podman sends SIGTERM to the container's main process (PID 1)
2. The process has until the timeout expires to shut down gracefully
3. If still running after the timeout, Podman sends SIGKILL

```bash
# Demonstrate the stop process with a container that handles SIGTERM
podman run -d --name graceful-stop docker.io/library/alpine:latest \
  sh -c 'trap "echo Received SIGTERM; exit 0" TERM; echo "Running..."; while true; do sleep 1; done'

# Give the container a moment to start
sleep 2

# Stop it and check the logs
podman stop -t 10 graceful-stop
podman logs graceful-stop
```

## Setting Default Stop Timeout During Creation

Set the stop timeout when creating the container so you do not need to specify it every time.

```bash
# Create a container with a custom stop timeout
podman run -d \
  --name slow-app \
  --stop-timeout 60 \
  docker.io/library/alpine:latest \
  sleep 3600

# This will now use the 60-second timeout
podman stop slow-app
```

## Setting the Stop Signal

Some applications listen for a different signal than SIGTERM. Configure this with `--stop-signal`.

```bash
# Create a container that stops on SIGINT instead of SIGTERM
podman run -d \
  --name custom-signal \
  --stop-signal SIGINT \
  docker.io/library/alpine:latest \
  sh -c 'trap "echo Got SIGINT; exit 0" INT; while true; do sleep 1; done'

sleep 2

# This sends SIGINT instead of SIGTERM
podman stop custom-signal
podman logs custom-signal

# Clean up
podman rm custom-signal
```

## Stop and Remove in One Step

If you want to stop and remove a container immediately:

```bash
# Stop and remove
podman stop my-nginx && podman rm my-nginx

# Or force-remove a running container (combines stop + rm)
podman rm -f my-nginx
```

## Scripting Graceful Shutdown

```bash
#!/bin/bash
# shutdown.sh - Gracefully shut down all application containers

echo "Stopping application containers..."

# Stop in reverse dependency order
SERVICES=("web" "api" "cache" "db")

for svc in "${SERVICES[@]}"; do
  if podman ps --format "{{.Names}}" | grep -q "^${svc}$"; then
    echo "Stopping $svc (30s timeout)..."
    podman stop -t 30 "$svc"
    echo "$svc stopped"
  else
    echo "$svc is not running"
  fi
done

echo "All services stopped"
podman ps --format "table {{.Names}}\t{{.Status}}"
```

## Summary

The `podman stop` command gracefully shuts down containers by sending SIGTERM followed by SIGKILL after a configurable timeout. Use `-t` to adjust the grace period, `--stop-signal` to change the shutdown signal, and `--all` to stop everything at once. Always prefer graceful stops over force-killing to give applications time to clean up.
