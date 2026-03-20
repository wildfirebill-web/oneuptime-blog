# How to View Service Task Status in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Services, Monitoring, DevOps

Description: Learn how to monitor individual Swarm service task status, placement, and health in Portainer.

## Introduction

In Docker Swarm, services are composed of individual tasks — each task is a container running on a specific node. Understanding task status is crucial for diagnosing deployment issues, monitoring rollouts, and verifying service health. Portainer provides detailed task views for all Swarm services. This guide covers how to read and act on task status information.

## Prerequisites

- Portainer installed on Docker Swarm
- At least one Swarm service running
- Admin or operator access

## Step 1: Navigate to Service Tasks

1. In Portainer, select your Swarm environment
2. Click **Services** in the sidebar
3. Click on the service name to open the detail view
4. Scroll down to the **Tasks** section

## Step 2: Read the Task Table

Each task shows:

| Column | Description |
|--------|-------------|
| **ID** | Short task identifier |
| **Slot** | Task slot number (e.g., 1, 2, 3) |
| **Node** | Hostname of the node running this task |
| **Status** | Current task state |
| **Image** | Docker image version for this task |
| **Last updated** | Timestamp of last state change |

## Task Status Values

| Status | Meaning |
|--------|---------|
| `running` | Task is healthy and running |
| `starting` | Task is being started |
| `pending` | Task waiting for resources or node |
| `complete` | Task completed (for one-shot tasks) |
| `failed` | Task exited with an error |
| `rejected` | Swarm rejected the task (e.g., constraint not met) |
| `shutdown` | Task was gracefully stopped |
| `orphaned` | Node with this task is unreachable |

## Step 3: Understand Task Slots

In replicated services, task slots persist:

```
Before update:
  web.1 → worker-01 (running)  ← Slot 1
  web.2 → worker-02 (running)  ← Slot 2
  web.3 → worker-03 (running)  ← Slot 3

During update (start-first order):
  web.1 → worker-01 (running, old)   ← Old task in slot 1
  web.1 → worker-02 (starting, new)  ← New task for slot 1
  web.2 → worker-02 (running, old)   ← Old task in slot 2
  web.3 → worker-03 (running, old)   ← Old task in slot 3
```

## Step 4: View Task History

Portainer shows current and historical tasks for the service. Historical tasks that have been shut down or failed appear with their status for debugging.

Filter to show only current tasks or include history using the filter options.

## Step 5: Click Into a Specific Task

Click on a running task ID to navigate to the container detail:

- **Logs** — View the container's stdout/stderr
- **Stats** — CPU, memory, network usage
- **Console** — Exec into the container
- **Inspect** — Full container JSON details

## Step 6: Diagnose Common Task States

### Task stuck in `pending`

```bash
# Check why a task is not scheduled
docker service ps --no-trunc web-frontend

# Common reasons:
# - No nodes satisfy placement constraints
# - Insufficient resources on all nodes
# - Image cannot be pulled
```

Check the **error** column in `docker service ps` for details:

```
no suitable node (scheduling constraints not satisfied on 4 nodes)
```

Resolution: Review constraints or add nodes that satisfy them.

### Task in `failed` state

```bash
# View failed task logs
docker service logs --no-trunc web-frontend

# Or check the specific task's exit code
docker inspect <task-id>
```

### Task repeatedly cycling (crash loop)

If you see many historical task entries for the same slot, the container is crash-looping:

```bash
# Service ps shows repeated failures
docker service ps web-frontend
# NAME         IMAGE        NODE       DESIRED STATE  CURRENT STATE
# web.1        myapp:v2     worker-01  Running        Running 10 seconds
# \_ web.1     myapp:v2     worker-01  Shutdown       Failed 25 seconds ago
# \_ web.1     myapp:v2     worker-01  Shutdown       Failed 55 seconds ago
# \_ web.1     myapp:v2     worker-02  Shutdown       Failed 2 minutes ago
```

This is a crash loop. Check application logs for the root cause.

### Task stuck in `orphaned`

The node running the task became unreachable:

```bash
# Check node status
docker node ls

# If node is permanently gone, force remove it
docker node rm --force <node-id>
```

## Step 7: Filtering Tasks in Portainer

Use Portainer's filter options to narrow the task view:

- **Show only running tasks** — Hide historical/failed tasks
- **Filter by state** — Focus on pending or failed tasks
- **Filter by node** — See tasks on a specific node

## CLI Reference for Task Inspection

```bash
# List all tasks for a service
docker service ps web-frontend

# List only failed tasks
docker service ps --filter desired-state=failed web-frontend

# Show detailed task info
docker inspect <task-id>

# Get the container ID from a task
docker inspect --format '{{.Status.ContainerStatus.ContainerID}}' <task-id>

# Access logs for a specific task's container
CONTAINER_ID=$(docker inspect --format '{{.Status.ContainerStatus.ContainerID}}' <task-id>)
docker logs $CONTAINER_ID
```

## Conclusion

Task status visibility is key to operating reliable Docker Swarm services. Portainer's task view gives you a clear picture of where each replica is running, its current health, and historical task information for debugging. By understanding the different task states and knowing how to diagnose each, you can quickly identify and resolve deployment issues in your Swarm cluster.
