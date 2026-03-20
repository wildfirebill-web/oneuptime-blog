# How to Set Up Log Rotation for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Log Rotation, Disk Management, Container Configuration, Maintenance

Description: Configure automatic log rotation for Docker containers to prevent disk exhaustion, using built-in json-file log driver options and global daemon settings in Portainer deployments.

## Introduction

Without log rotation, container log files grow indefinitely until they fill the disk. Docker's default `json-file` log driver supports built-in log rotation via `max-size` and `max-file` options. Configuring these prevents disk exhaustion while retaining enough recent logs for troubleshooting. This guide covers configuring log rotation globally and per-service in Portainer-managed environments.

## Step 1: Configure Global Log Rotation

```json
// /etc/docker/daemon.json - Apply log rotation to all containers
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",    // Max size per log file
    "max-file": "3",      // Keep 3 rotated files (30MB total per container)
    "compress": "true"    // Compress rotated files with gzip
  }
}
```

```bash
# Apply the changes

sudo systemctl restart docker

# Verify default log options
docker info | grep -A 5 "Logging Driver"

# IMPORTANT: This only affects NEW containers
# Existing containers keep their current log configuration
# Recreate containers to apply the new settings
```

## Step 2: Configure Log Rotation Per Service

Per-service settings override the daemon default:

```yaml
# docker-compose.yml - Log rotation per service
version: "3.8"

services:
  # High-volume API: smaller files, more rotations
  api:
    image: myapp/api:latest
    logging:
      driver: json-file
      options:
        max-size: "20m"     # 20MB per file
        max-file: "5"       # Keep 5 files = 100MB max
        compress: "true"    # Gzip rotated files (saves ~70% space)
        # Add container labels to log metadata
        labels: "service,environment"
        env: "NODE_ENV"

  # Database: verbose but important logs
  postgres:
    image: postgres:15-alpine
    logging:
      driver: json-file
      options:
        max-size: "50m"     # Larger files for DB logs
        max-file: "10"      # Keep 10 files = 500MB max
        compress: "true"

  # Redis: low volume, minimal retention
  redis:
    image: redis:7-alpine
    logging:
      driver: json-file
      options:
        max-size: "5m"
        max-file: "2"      # Only 10MB total for Redis
        compress: "true"

  # Nginx: access logs can be very high volume
  nginx:
    image: nginx:alpine
    logging:
      driver: json-file
      options:
        max-size: "100m"    # Nginx access logs are large
        max-file: "5"       # 500MB max for nginx
        compress: "true"
```

## Step 3: Calculate Appropriate Rotation Settings

```bash
# Check current log sizes to understand your baseline
du -sh /var/lib/docker/containers/*/
# or summarized:
du -sh /var/lib/docker/containers/*/*.log | sort -h | tail -20

# Check total Docker log disk usage
docker system df -v | grep "CONTAINER\|LOG SIZE"

# Calculate log growth rate
CONTAINER_ID=$(docker ps -q --filter name=api)
LOG_PATH="/var/lib/docker/containers/$CONTAINER_ID/$CONTAINER_ID-json.log"

SIZE_BEFORE=$(stat -c%s "$LOG_PATH")
sleep 3600  # Wait 1 hour
SIZE_AFTER=$(stat -c%s "$LOG_PATH")
GROWTH_PER_HOUR=$(( (SIZE_AFTER - SIZE_BEFORE) / 1024 / 1024 ))

echo "API log growth: ${GROWTH_PER_HOUR}MB/hour"
echo "Daily growth: $(( GROWTH_PER_HOUR * 24 ))MB/day"
# Set max-size to ~2x hourly growth rate for useful troubleshooting window
```

## Step 4: Monitor Log File Sizes

```bash
#!/bin/bash
# monitor-log-sizes.sh - Alert on large container logs

MAX_LOG_SIZE_MB=200  # Alert threshold in MB

docker ps -q | while read id; do
  name=$(docker inspect "$id" --format '{{.Name}}')
  log_path=$(docker inspect "$id" \
    --format '/var/lib/docker/containers/{{.Id}}/{{.Id}}-json.log')

  if [ -f "$log_path" ]; then
    size_mb=$(du -m "$log_path" | cut -f1)

    if [ "$size_mb" -gt "$MAX_LOG_SIZE_MB" ]; then
      echo "ALERT: $name log file is ${size_mb}MB (threshold: ${MAX_LOG_SIZE_MB}MB)"
      echo "       Log path: $log_path"
    fi
  fi
done

# Check total Docker disk usage
TOTAL=$(du -sh /var/lib/docker/containers/ | cut -f1)
echo "Total container log storage: $TOTAL"
```

## Step 5: Truncate Logs Without Restarting Containers

For emergency disk recovery without stopping containers:

```bash
# SAFE: Truncate a specific container's log (Docker reopens the file)
CONTAINER_ID=$(docker ps -q --filter name=api)
LOG_PATH="/var/lib/docker/containers/$CONTAINER_ID/$CONTAINER_ID-json.log"

# Truncate to 0 bytes (container keeps running, just loses old logs)
truncate -s 0 "$LOG_PATH"

# Or truncate to keep only last 1000 lines
LOG_LINES=$(wc -l < "$LOG_PATH")
KEEP_LINES=1000
if [ $LOG_LINES -gt $KEEP_LINES ]; then
  tail -n $KEEP_LINES "$LOG_PATH" > /tmp/log.tmp && mv /tmp/log.tmp "$LOG_PATH"
fi

# For all containers at once (emergency disk recovery)
for id in $(docker ps -q); do
  log="/var/lib/docker/containers/$id/$id-json.log"
  if [ -f "$log" ]; then
    size=$(du -m "$log" | cut -f1)
    if [ "$size" -gt 100 ]; then
      echo "Truncating $log (${size}MB)"
      truncate -s 0 "$log"
    fi
  fi
done
```

## Step 6: Logrotate for Additional Control

```bash
# /etc/logrotate.d/docker-containers
# Additional logrotate config for Docker container logs

/var/lib/docker/containers/**/*-json.log {
  rotate 5
  daily
  compress
  delaycompress
  missingok
  notifempty
  sharedscripts
  copytruncate  # Truncate in-place so Docker doesn't lose file handle
  size 50M      # Rotate when file exceeds 50MB (in addition to daily)
  postrotate
    # Optional: Signal Docker to reopen log files
    # docker ps -q | xargs -I{} docker kill --signal SIGHUP {}
  endscript
}
```

```bash
# Test logrotate configuration
sudo logrotate -d /etc/logrotate.d/docker-containers
# -d = dry run (shows what would happen)

# Force rotation now
sudo logrotate -f /etc/logrotate.d/docker-containers
```

## Conclusion

Log rotation is essential disk management for containerized environments. Docker's built-in `max-size` and `max-file` options in the json-file driver are the simplest approach - no external tools needed. Set them globally in `daemon.json` for consistent behavior, then override per-service for high-volume containers. The `compress: "true"` option is often overlooked but typically saves 60-80% of log storage for text-based logs. Monitor log sizes regularly and tune the rotation thresholds based on observed growth rates. Portainer's compose YAML makes it straightforward to apply different log retention policies to different services based on their criticality and volume.
