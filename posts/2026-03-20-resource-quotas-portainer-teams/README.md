# How to Configure Per-Team Resource Quotas in Portainer - Teams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Resource Quota, Multi-Tenant, Team, Resource Management

Description: Set CPU, memory, and container count limits per team in Portainer to prevent resource starvation between tenants sharing the same Docker infrastructure.

## Introduction

When multiple teams share a Docker host, resource quotas prevent one team from consuming all available CPU and memory, starving other teams' workloads. Portainer Business Edition supports resource quotas at the environment level and container-level limits for team users. This guide covers configuring resource limits at both the infrastructure and Portainer policy levels.

## Step 1: Set Container Resource Limits in Team Stacks

```yaml
# Team Alpha's docker-compose.yml with enforced resource limits

version: "3.8"

services:
  api:
    image: alpha/api:latest
    deploy:
      resources:
        limits:
          cpus: "1.0"      # Max 1 CPU
          memory: 512M     # Max 512MB RAM
        reservations:
          cpus: "0.25"     # Guaranteed 0.25 CPU
          memory: 128M     # Guaranteed 128MB RAM
    mem_limit: 512m         # For non-Swarm deployments
    mem_reservation: 128m
    cpus: 1.0

  database:
    image: postgres:15-alpine
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
    mem_limit: 2g
    environment:
      - POSTGRES_SHARED_BUFFERS=512MB  # Postgres respects memory limit

  worker:
    image: alpha/worker:latest
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    mem_limit: 256m
```

## Step 2: Portainer Security Settings to Enforce Quotas

```bash
PORTAINER_URL="https://portainer.example.com"
ADMIN_TOKEN="admin_token"
ENV_ID=1  # The environment team uses

# Configure security settings that prevent teams from bypassing limits
curl -s -X PUT \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/endpoints/$ENV_ID" \
  -d '{
    "SecuritySettings": {
      "allowBindMountsForRegularUsers": false,
      "allowPrivilegedModeForRegularUsers": false,
      "allowHostNamespaceForRegularUsers": false,
      "allowDeviceMappingForRegularUsers": false,
      "allowSysctlSettingForRegularUsers": false,
      "allowContainerCapabilitiesForRegularUsers": false,
      "enableHostManagementFeatures": false
    }
  }'

# With these settings, team users CANNOT:
# - Deploy containers without resource limits
# - Run privileged containers
# - Mount host filesystem paths
# - Modify kernel parameters
```

## Step 3: cgroups-Based Host-Level Quotas

For strict enforcement at the OS level, use cgroups:

```bash
# Create cgroup hierarchy for tenant isolation
# Linux cgroups v2

# Create cgroup for Team Alpha
mkdir -p /sys/fs/cgroup/team-alpha

# Set CPU quota (50% of one CPU)
echo "50000 100000" > /sys/fs/cgroup/team-alpha/cpu.max
# Format: quota_microseconds period_microseconds
# 50000/100000 = 50% of one CPU

# Set memory limit (4GB)
echo "4294967296" > /sys/fs/cgroup/team-alpha/memory.max  # 4GB in bytes

# Set swap limit (no swap)
echo "4294967296" > /sys/fs/cgroup/team-alpha/memory.swap.max

# Docker can place containers in specific cgroups
docker run -d \
  --cgroup-parent /team-alpha \
  --name alpha-api \
  alpha/api:latest
```

## Step 4: Monitor Resource Usage Per Team

```bash
#!/bin/bash
# team-resource-report.sh - Show resource usage by team label

echo "=== Team Resource Usage Report ==="
echo "Date: $(date)"
echo ""

# Get stats for Team Alpha containers
echo "--- Team Alpha ---"
docker stats --no-stream \
  $(docker ps --filter "label=tenant=alpha" -q) \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

echo ""
echo "--- Team Beta ---"
docker stats --no-stream \
  $(docker ps --filter "label=tenant=beta" -q) \
  --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Aggregate totals
echo ""
echo "--- Resource Totals ---"

# Sum CPU usage for each team
ALPHA_CPU=$(docker stats --no-stream \
  $(docker ps --filter "label=tenant=alpha" -q) \
  --format "{{.CPUPerc}}" | \
  awk '{sum += $1} END {printf "%.1f%%", sum}')

BETA_CPU=$(docker stats --no-stream \
  $(docker ps --filter "label=tenant=beta" -q) \
  --format "{{.CPUPerc}}" | \
  awk '{sum += $1} END {printf "%.1f%%", sum}')

echo "Team Alpha CPU: $ALPHA_CPU"
echo "Team Beta CPU: $BETA_CPU"
```

## Step 5: Alert on Resource Quota Violations

```bash
#!/bin/bash
# quota-alert.sh - Alert when team containers exceed limits

MAX_MEMORY_MB=512   # Alert if any container uses more than 512MB
MAX_CPU_PERCENT=80  # Alert if any container uses more than 80% CPU

docker stats --no-stream --format "{{.Name}} {{.MemUsage}} {{.CPUPerc}}" | \
while read name mem_usage mem_total cpu_perc; do
  # Extract memory in MB
  mem_mb=$(echo "$mem_usage" | sed 's/MiB//' | awk '{print int($1)}')
  cpu_num=$(echo "$cpu_perc" | tr -d '%' | awk '{print int($1)}')

  if [ "$mem_mb" -gt "$MAX_MEMORY_MB" ] 2>/dev/null; then
    echo "ALERT: $name is using ${mem_mb}MB (limit: ${MAX_MEMORY_MB}MB)"
    # Send alert to Slack/PagerDuty
  fi

  if [ "$cpu_num" -gt "$MAX_CPU_PERCENT" ] 2>/dev/null; then
    echo "ALERT: $name CPU at ${cpu_num}% (threshold: ${MAX_CPU_PERCENT}%)"
  fi
done
```

## Step 6: Enforce Limits via Portainer Stack Templates

```yaml
# Create a stack template that enforces limits
# Teams deploy from this template, ensuring limits are always set

# portainer-template-api.yml
version: "3.8"

services:
  app:
    image: "${APP_IMAGE}"
    environment:
      - NODE_ENV=production
    deploy:
      resources:
        limits:
          # These values come from Portainer template variables
          cpus: "${CPU_LIMIT:-0.5}"
          memory: "${MEMORY_LIMIT:-256M}"
    mem_limit: "${MEMORY_LIMIT:-256m}"
    restart: unless-stopped
    labels:
      - "tenant=${TEAM_NAME}"
```

```bash
# Create the template in Portainer
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  "$PORTAINER_URL/api/templates" \
  -d '{
    "Title": "Application Stack with Resource Limits",
    "Description": "Deploy an application with enforced CPU and memory limits",
    "Type": 2,
    "Variables": [
      {"name": "TEAM_NAME", "label": "Team Name", "default": ""},
      {"name": "APP_IMAGE", "label": "Docker Image", "default": ""},
      {"name": "CPU_LIMIT", "label": "CPU Limit", "default": "0.5"},
      {"name": "MEMORY_LIMIT", "label": "Memory Limit", "default": "256m"}
    ]
  }'
```

## Conclusion

Resource quotas in a shared Portainer environment require enforcement at multiple levels: Docker compose `mem_limit` and `cpus` settings for individual containers, Portainer security policies to prevent teams from deploying without limits, and optional OS-level cgroup quotas for hard enforcement. Monitoring scripts that track resource usage by team label provide visibility into consumption patterns. Stack templates with required resource variables ensure new deployments always include limits, preventing accidental resource exhaustion that impacts other teams.
