# How to Pause and Unpause Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Operations, DevOps

Description: Learn how to pause and unpause Docker containers in Portainer to temporarily freeze container execution without stopping or losing its state.

## Introduction

Pausing a container is different from stopping it. When paused, the container's processes are frozen using Linux's `cgroup freezer` — they stop executing but remain in memory with all their state intact. Unpausing resumes execution exactly where it left off. This is useful for maintenance, debugging, and performance isolation.

## Prerequisites

- Portainer installed with a connected Docker environment
- Running containers

## How Docker Pause Works

The pause mechanism uses the Linux `cgroup freezer` subsystem:

```
Normal state:     Container processes executing → cgroup freezer: THAWED
Paused state:     Container processes frozen  → cgroup freezer: FROZEN
Unpaused state:   Container processes resume  → cgroup freezer: THAWED
```

When paused:
- Processes are suspended (no CPU time allocated).
- Memory contents are preserved.
- Open file handles remain open.
- Network connections don't receive data but are not closed.
- The container appears as **Paused** in Portainer.

## Step 1: Pause a Container

### Via the Container List

1. Navigate to **Containers** in Portainer.
2. Find the running container.
3. Click the **Pause** button (two vertical bars icon).

### Via the Container Details Page

1. Click on the container name.
2. Look for the **Pause** button in the action bar.

```bash
# Equivalent Docker CLI:
docker pause my-container

# Verify the container is paused:
docker ps | grep my-container
# Status shows: Up X minutes (Paused)
```

## Step 2: Unpause a Container

### Via the Container List

1. Find the paused container (shown with "Paused" status).
2. Click the **Unpause** button (or Resume button).

### Via the Container Details Page

1. Click on the paused container.
2. Click **Unpause**.

```bash
# Equivalent Docker CLI:
docker unpause my-container
```

## When to Use Pause vs. Stop

| Scenario | Action | Why |
|----------|--------|-----|
| Temporary maintenance | **Pause** | Fast resume, no data loss |
| Reducing load during peak hours | **Pause** | Release CPU without losing state |
| Debugging a running container's state | **Pause** | Freeze state for inspection |
| Taking a consistent snapshot/backup | **Pause** | Ensure data consistency |
| Shutting down permanently | **Stop** | Clean shutdown with SIGTERM |
| Applying configuration changes | **Recreate** | New container with new config |

## Use Case 1: Database Backup with Consistent State

Pause a database container to take a consistent backup without stopping it:

```bash
#!/bin/bash
# Backup script: pause container, backup volume, unpause

CONTAINER="postgres-db"
BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Pause the container to get a consistent state
echo "Pausing ${CONTAINER}..."
docker pause "${CONTAINER}"

# Back up the data volume
echo "Backing up data..."
tar -czf "${BACKUP_DIR}/db-backup-${TIMESTAMP}.tar.gz" \
    -C /var/lib/docker/volumes/myapp_postgres_data/_data .

# Resume the container
echo "Unpausing ${CONTAINER}..."
docker unpause "${CONTAINER}"

echo "Backup complete: db-backup-${TIMESTAMP}.tar.gz"
```

In Portainer:
1. Navigate to the database container.
2. Click **Pause**.
3. Perform your backup operation.
4. Click **Unpause**.

## Use Case 2: Performance Isolation

Temporarily pause non-critical containers during a high-load event:

```bash
#!/bin/bash
# pause-non-critical.sh
# Pause background workers during peak hours

NON_CRITICAL_CONTAINERS=(
  "report-generator"
  "email-digest-sender"
  "analytics-processor"
)

for container in "${NON_CRITICAL_CONTAINERS[@]}"; do
  echo "Pausing ${container}..."
  docker pause "${container}" 2>/dev/null || echo "  ${container} not running"
done

echo "Non-critical containers paused. Run unpause-non-critical.sh to resume."
```

## Use Case 3: Development Debugging

Freeze a container at a specific moment to inspect its state:

```bash
# Pause a container mid-execution
docker pause my-app

# Inspect its state while frozen
# In Portainer: view logs, inspect, network connections all still visible

# Examine network connections from the host:
ss -tlnp | grep docker

# Check memory usage while frozen:
docker stats --no-stream my-app

# Resume when done debugging
docker unpause my-app
```

## Step 3: Bulk Pause/Unpause in Portainer

For multiple containers:

1. Navigate to **Containers**.
2. Check the checkboxes next to the containers.
3. Click **Pause** or **Unpause** in the bulk action bar.

## Monitoring Paused Containers

In Portainer, paused containers:
- Show **Paused** status in the container list.
- Still consume memory (processes remain in RAM).
- Do not consume CPU.
- Are not removed by `docker container prune`.

```bash
# List paused containers:
docker ps --filter status=paused

# Check container state:
docker inspect my-container | jq '.[].State.Paused'
# Returns: true
```

## Important Limitations

- **Paused containers still consume memory** — don't pause as a memory optimization.
- **Network connections may time out** — if paused too long, TCP connections may be dropped by the peer.
- **Healthchecks fail** — paused containers fail health checks, which may trigger restart policies.
- **Not suitable for long-term suspension** — use stop for anything longer than a few minutes.

## Conclusion

The pause/unpause functionality in Portainer provides a way to temporarily freeze container execution without losing state. It's most valuable for ensuring data consistency during backups, reducing CPU contention during peak periods, and debugging container behavior. For anything requiring a longer interruption, stop the container instead — paused containers still hold memory and may have connection issues after an extended pause.
