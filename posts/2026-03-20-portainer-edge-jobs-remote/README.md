# How to Run Edge Jobs Across Remote Environments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Edge Computing, Automation, DevOps

Description: Learn how to schedule and run one-off or recurring Edge Jobs across remote edge environments using Portainer's Edge Compute features.

## Introduction

Edge Jobs in Portainer allow you to run a script or command on a Docker container across one or many remote edge devices - on a schedule or on demand. This is ideal for tasks like log rotation, health checks, certificate renewals, database backups, or any maintenance task that needs to run on edge hardware.

## Prerequisites

- Portainer Business Edition with Edge Compute enabled
- Edge agents connected and assigned to edge groups
- Admin or edge admin role in Portainer

## What Are Edge Jobs?

An Edge Job is essentially a scheduled container task (like a Docker `run` with a command). Portainer:
1. Sends the job definition to the edge agents.
2. Each agent runs the job in a temporary container.
3. Results and logs are collected and sent back to Portainer.

Edge Jobs support both **on-demand** (run once) and **cron-based** (scheduled) execution.

## Step 1: Navigate to Edge Jobs

1. Log in to Portainer.
2. Go to **Edge Compute > Edge Jobs**.
3. Click **Add Edge Job**.

## Step 2: Configure the Edge Job

Fill in the form:

- **Name**: A descriptive label (e.g., `log-rotation-nightly`).
- **Run mode**:
  - **Recurring** - provide a cron expression.
  - **Once** - runs as soon as agents receive the job.

```text
# Cron expression examples:

# Run every night at 2:00 AM
0 2 * * *

# Run every hour
0 * * * *

# Run every Monday at 8:00 AM
0 8 * * 1
```

## Step 3: Write the Job Script

The script runs inside a container. Use a base image appropriate for your task.

```bash
#!/bin/sh
# Example: clean up logs older than 7 days on the host
# This runs inside a container with /host-logs mounted

find /host-logs -name "*.log" -mtime +7 -delete
echo "Log cleanup complete on $(hostname)"
```

Or a more advanced backup job:

```bash
#!/bin/sh
# Example: back up a SQLite database to a remote S3 bucket

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DEVICE_ID=${DEVICE_ID:-unknown}

# Copy DB file and upload to S3
cp /data/app.db /tmp/app_backup_${TIMESTAMP}.db
aws s3 cp /tmp/app_backup_${TIMESTAMP}.db \
    s3://mybucket/backups/${DEVICE_ID}/app_backup_${TIMESTAMP}.db

echo "Backup complete: ${DEVICE_ID} at ${TIMESTAMP}"
```

## Step 4: Select the Container Image

Choose an image that has the required tools. Examples:

```yaml
# For shell/cron jobs
image: alpine:latest

# For AWS CLI jobs
image: amazon/aws-cli:latest

# For custom maintenance containers
image: myorg/maintenance-tools:1.0
```

In Portainer's UI, enter the image name in the **Image** field of the Edge Job form.

## Step 5: Configure Volume Mounts (Optional)

If your job needs access to host files or Docker volumes, configure bind mounts:

```text
Host path:       /var/log/myapp
Container path:  /host-logs
Mode:            Read/Write
```

This allows your job script to read or write files on the edge device's filesystem.

## Step 6: Target Edge Groups or Endpoints

Under **Edge Groups** or **Endpoints**, select where the job should run:

- Select a full edge group (e.g., `Factory-Floor-Berlin`) to run on all devices.
- Select individual endpoints for targeted execution.

## Step 7: Review Job Results

After the job runs:
1. Navigate to **Edge Compute > Edge Jobs**.
2. Click on the job name.
3. View **Last execution** results per endpoint, including:
   - Exit code
   - Standard output/error from the container

```text
# Example output in Portainer Edge Job results:
Endpoint: device-berlin-001  | Status: Completed | Exit Code: 0
Output: Log cleanup complete on device-berlin-001
Backup complete: device-berlin-001 at 20260320_020001
```

## Best Practices

- **Use lightweight images** (e.g., `alpine`) for simple shell scripts to minimize download time on edge devices.
- **Log output to stdout/stderr** so Portainer captures it in the job results.
- **Test jobs on a single endpoint** before targeting an entire edge group.
- **Set sensible cron schedules** - avoid running heavy jobs during peak operational hours.
- **Idempotent scripts**: Design scripts so that re-running them doesn't cause harm.

## Conclusion

Portainer Edge Jobs provide a simple, centralized way to run administrative and maintenance tasks across your entire edge fleet. With cron scheduling and per-endpoint result tracking, you get the visibility and control needed to keep distributed environments running smoothly without direct SSH access to each device.
