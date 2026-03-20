# How to Restart Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Operations, DevOps

Description: Learn how to restart Docker containers in Portainer, including graceful restarts, bulk restarts, and when to use restart versus recreate.

## Introduction

Restarting a container is a common operation — needed after configuration changes, to recover from a hung state, or to apply log rotation. Portainer provides direct restart controls in the container list and details view. This guide covers when and how to restart containers, and the difference between restart, stop+start, and recreate.

## Prerequisites

- Portainer installed with a connected Docker environment
- Running containers to manage

## How Docker Restart Works

`docker restart` performs:
1. Sends `SIGTERM` to the container process.
2. Waits for the grace period (default: 10 seconds).
3. Sends `SIGKILL` if the process hasn't stopped.
4. Starts the container again from the same configuration.

The container retains all its settings (volumes, environment, ports) — it just restarts the process.

## Step 1: Restart a Single Container

### Via the Container List

1. Navigate to **Containers** in Portainer.
2. Find the container to restart.
3. Click the **Restart** button (circular arrow icon).

### Via the Container Details Page

1. Click on the container name.
2. Click the **Restart** button at the top.

```bash
# Equivalent Docker CLI:
docker restart my-container

# With custom timeout:
docker restart --time 30 my-container
```

## Step 2: Restart Multiple Containers

1. Navigate to **Containers**.
2. Check the checkboxes next to containers to restart.
3. Click **Restart** in the bulk action bar.

## Step 3: Scheduling Automatic Restarts

### Using Cron Jobs on the Host

For scheduled restarts (e.g., nightly restart to clear memory leaks):

```bash
#!/bin/bash
# nightly-restart.sh
# Scheduled via cron: 0 2 * * * /opt/scripts/nightly-restart.sh

containers=(
  "my-app-1"
  "my-app-2"
  "web-server"
)

for container in "${containers[@]}"; do
  echo "Restarting ${container}..."
  docker restart "${container}"
  echo "Restarted: ${container}"
done
```

### Using Portainer Edge Jobs for Edge Devices

For edge devices, create an Edge Job that restarts containers:

```bash
#!/bin/sh
# Edge job: restart application container
docker restart app-container
echo "Container restarted successfully"
```

## Step 4: Restart vs. Stop + Start vs. Recreate

| Operation | What it does | When to use |
|-----------|-------------|-------------|
| **Restart** | Stops and starts the same container | Config reload, memory flush |
| **Stop + Start** | Same as restart, but in two steps | When you need a pause between stop and start |
| **Recreate** | Removes old container, creates new one | When you change container configuration (ports, env vars, volumes) |
| **Re-pull + Recreate** | Pulls new image, then recreates | When deploying a new version of the image |

### When Restart Is Sufficient

```bash
# Restart refreshes the process without losing container settings
# Good for:
# - Application hung or memory leak
# - Config file reloaded inside the container
# - Recovering from a temporary error
docker restart my-app
```

### When to Recreate Instead

```bash
# Recreate is needed when:
# - You changed environment variables
# - You changed port mappings
# - You changed volume mounts
# - You changed the image

# In Portainer: use "Duplicate/Edit" then "Deploy"
# This creates a new container with updated settings
```

## Step 5: Automated Restart via Portainer Webhooks

You can trigger a container restart via Portainer's webhook feature:

```bash
# Trigger a container redeploy (pull + restart) via webhook
curl -X POST "https://portainer.example.com/api/webhooks/your-webhook-token"

# In CI/CD pipeline:
- name: Restart app container
  run: |
    curl -X POST "${{ secrets.PORTAINER_WEBHOOK_URL }}"
```

## Step 6: Health-Check-Triggered Restarts

Configure Docker to automatically restart containers that fail health checks:

```yaml
# docker-compose.yml
services:
  app:
    image: myorg/myapp:latest
    restart: unless-stopped   # Restart policy for auto-restart on crash
    healthcheck:
      # Docker restarts the container if health check consistently fails
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

When the healthcheck fails `retries` times, Docker marks the container as `unhealthy`. Combined with the restart policy, Docker will restart it automatically.

## Step 7: Monitoring Restart Behavior

After restarting, verify the container is running properly:

```bash
# Check container status:
docker inspect my-container | jq '.[].State'

# In Portainer: navigate to container details
# Check:
# - Status: Running
# - RestartCount: number of restarts
# - StartedAt: last start time
```

Portainer shows the restart count on the container details page — a high count indicates a recurring problem.

## Best Practices

- **Use health checks** to enable automatic recovery without manual intervention.
- **Investigate before restarting in a loop** — a container that keeps restarting has a bug that needs fixing.
- **Use `restart: unless-stopped`** for production services so they auto-recover.
- **Restart before investigating memory issues** — a restart confirms whether the issue is a memory leak that's fixed by restart, or a deeper configuration problem.
- **Schedule restarts during maintenance windows** for non-high-availability services that benefit from periodic restarts.

## Conclusion

Restarting containers in Portainer is straightforward and covers most operational needs. For configuration changes that require modifying the container itself (environment variables, volumes), use the recreate approach instead. Combine Docker's built-in health checks with restart policies to minimize manual interventions and keep your services running automatically.
