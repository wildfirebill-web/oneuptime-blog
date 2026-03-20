# How to Monitor Disk Space per Volume in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Monitoring, Storage

Description: Learn how to monitor disk space usage per volume in Portainer to prevent storage exhaustion and keep your containerized workloads healthy.

## Introduction

Running containers without monitoring disk space can lead to unexpected failures, data loss, or degraded application performance. Portainer provides built-in tools to inspect volume usage, and combined with host-level commands, you can maintain full visibility into your storage consumption.

## Prerequisites

- Portainer CE or BE installed and running
- At least one Docker environment connected to Portainer
- Admin or operator-level access

## Viewing Volume Usage in Portainer UI

### Step 1: Navigate to Volumes

1. Log in to your Portainer instance.
2. From the left sidebar, select your **Environment** (e.g., local Docker).
3. Click on **Volumes** in the left navigation menu.

You will see a list of all volumes with their names, mount points, driver, and creation date.

### Step 2: Inspect an Individual Volume

Click on any volume name to open its details page. Here you can see:

- **Driver**: e.g., `local`
- **Mount Point**: the path on the host where the data lives
- **Containers using this volume**: which containers are currently attached

> Note: Portainer does not show real-time disk usage numbers in the volume list by default. Use the methods below to get actual byte consumption.

## Checking Disk Usage from the Container Console

You can open a shell inside a running container that uses the volume and inspect the filesystem:

```bash
# Inside the container shell (open via Portainer > Container > Console)

df -h /data

# Example output:
# Filesystem      Size  Used Avail Use% Mounted on
# overlay         100G   12G   88G  12% /
# /dev/sda1        50G   18G   32G  36% /data
```

## Using the Portainer Exec / Console Feature

1. Go to **Containers** and click your container.
2. Click **Console** → **Connect**.
3. Run `df -h` to see all mount points and their usage.

## Monitoring Disk Usage via the Docker API (via Portainer API)

You can also query disk usage programmatically using the Docker System API routed through Portainer:

```bash
# Authenticate and get JWT token
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Get disk usage info for environment ID 1
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1/docker/system/df" | jq .

# The Volumes array in the response includes UsageData.Size per volume
```

The response includes a `Volumes` array where each entry has a `UsageData.Size` field showing byte consumption.

## Automating Volume Usage Alerts

You can write a simple monitoring script that alerts when a volume exceeds a threshold:

```bash
#!/bin/bash
# monitor-volumes.sh - Alert when volume usage exceeds threshold

PORTAINER_URL="https://portainer.example.com"
PORTAINER_TOKEN="your-jwt-token"
ENDPOINT_ID=1
THRESHOLD_GB=10  # Alert if volume exceeds 10 GB

# Fetch disk usage data
USAGE=$(curl -s -H "Authorization: Bearer $PORTAINER_TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/system/df")

# Parse volumes and check size (size is in bytes)
echo "$USAGE" | jq -r '.Volumes[] | "\(.Name) \(.UsageData.Size)"' | while read name size; do
  size_gb=$(echo "scale=2; $size / 1073741824" | bc)
  if (( $(echo "$size_gb > $THRESHOLD_GB" | bc -l) )); then
    echo "ALERT: Volume $name is using ${size_gb}GB (threshold: ${THRESHOLD_GB}GB)"
    # Add notification logic here (e.g., curl to Slack webhook)
  fi
done
```

## Using Host-Level Commands

If you have SSH access to the Docker host, you can check actual volume directory sizes:

```bash
# List all Docker volumes with their disk usage on the host
sudo du -sh /var/lib/docker/volumes/*

# Example output:
# 2.1G   /var/lib/docker/volumes/myapp_postgres_data
# 512M   /var/lib/docker/volumes/myapp_redis_data
# 50M    /var/lib/docker/volumes/myapp_uploads

# Check a specific volume
sudo du -sh /var/lib/docker/volumes/myapp_postgres_data/_data
```

## Setting Up Ongoing Monitoring with Prometheus

For production environments, deploy cAdvisor and Prometheus to collect volume metrics automatically. You can then set alerts in Grafana or AlertManager when storage usage grows beyond defined thresholds.

```yaml
# Add to your Prometheus alerting rules
groups:
  - name: docker_volume_alerts
    rules:
      - alert: VolumeHighUsage
        expr: container_fs_usage_bytes{volume!=""} > 10737418240  # 10GB
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Docker volume {{ $labels.volume }} is using {{ $value | humanize }}B"
```

## Conclusion

Monitoring disk space per volume in Portainer is essential for maintaining reliable container workloads. Use the Portainer UI for quick inspections, the Docker API for programmatic access, and host-level commands for precise byte counts. For production environments, integrate Prometheus and AlertManager to receive proactive alerts before storage issues impact your services.
