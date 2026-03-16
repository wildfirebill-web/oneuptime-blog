# How to Configure Log File Rotation in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Logging, Configuration, Disk Management

Description: Learn how to configure automatic log file rotation in Podman using max-size and max-file options to manage disk space while retaining enough log history for debugging.

---

> Log rotation is the mechanism that prevents your logs from growing forever. In Podman, it is controlled by two simple options: max-size and max-file.

Without log rotation, container log files grow unbounded until they consume all available disk space. Podman supports built-in log rotation through the `max-size` and `max-file` log options, which work with the k8s-file and json-file log drivers.

---

## How Log Rotation Works in Podman

When log rotation is configured, Podman does the following:

1. Writes logs to the primary log file.
2. When the file reaches `max-size`, it renames the current file and creates a new one.
3. When the number of files exceeds `max-file`, the oldest file is deleted.

```bash
# Example: max-size=10m, max-file=3 creates:
# container-log         (current, up to 10MB)
# container-log.1       (previous, 10MB)
# container-log.2       (oldest, 10MB)
# Total maximum: 30MB
```

## Configure Rotation Per Container

```bash
# Basic rotation: 10MB file, keep 3 files
podman run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name web \
  nginx:latest

# Larger rotation for high-volume services
podman run -d \
  --log-opt max-size=100m \
  --log-opt max-file=5 \
  --name api \
  my-api:latest

# Minimal rotation for low-priority containers
podman run -d \
  --log-opt max-size=5m \
  --log-opt max-file=2 \
  --name cache \
  redis:latest

# Verify rotation settings
podman inspect --format '{{json .HostConfig.LogConfig}}' web | python3 -m json.tool
```

## Verify Rotation Is Working

```bash
# Generate logs to trigger rotation
podman run -d \
  --log-opt max-size=1m \
  --log-opt max-file=3 \
  --name rotation-test \
  alpine sh -c 'while true; do echo "$(date) - This is a log line with some padding to fill up space quickly: $(head -c 200 /dev/urandom | base64)"; sleep 0.01; done'

# Wait a moment, then check the log files
sleep 30
LOG_PATH=$(podman inspect --format '{{.LogPath}}' rotation-test)
ls -lh "${LOG_PATH}"*

# You should see multiple files:
# -rw-r--r-- 1 user user 980K container-json.log
# -rw-r--r-- 1 user user 1.0M container-json.log.1
# -rw-r--r-- 1 user user 1.0M container-json.log.2

# Clean up
podman rm -f rotation-test
```

## Configure System-Wide Rotation Defaults

Set default rotation for all new containers.

```bash
# For rootless Podman
mkdir -p ~/.config/containers

cat > ~/.config/containers/containers.conf << 'EOF'
[containers]
log_size_max = 10485760
EOF

# log_size_max is in bytes (10485760 = 10MB)
# Note: max-file default is typically 5

# For system-wide (root) configuration
# sudo vi /etc/containers/containers.conf

# Verify
podman info --format '{{.Host.LogSizeMax}}'
```

## Rotation with Different Log Drivers

```bash
# k8s-file driver rotation
podman run -d \
  --log-driver k8s-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name k8s-test \
  alpine sh -c 'while true; do echo "k8s log line"; sleep 0.1; done'

# json-file driver rotation
podman run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name json-test \
  alpine sh -c 'while true; do echo "json log line"; sleep 0.1; done'

# journald rotation (managed by journald, not Podman)
# Configure in /etc/systemd/journald.conf:
# SystemMaxUse=500M
# SystemMaxFileSize=50M
# MaxRetentionSec=1week
sudo systemctl restart systemd-journald
```

## Reading Rotated Log Files

When using `podman logs`, Podman automatically reads across rotated files.

```bash
# podman logs reads all rotated files seamlessly
podman logs web

# To read raw rotated files directly:
LOG_PATH=$(podman inspect --format '{{.LogPath}}' web)

# Read all rotated files in order (oldest first)
for f in $(ls -r "${LOG_PATH}"*); do
  cat "$f"
done

# Count total lines across all log files
cat "${LOG_PATH}"* 2>/dev/null | wc -l
```

## Monitor Rotation Health

```bash
#!/bin/bash
# check-log-rotation.sh - Verify rotation is working for all containers

echo "Container Log Rotation Status"
echo "=============================="

for c in $(podman ps --format '{{.Names}}'); do
  LOG_PATH=$(podman inspect --format '{{.LogPath}}' "$c" 2>/dev/null)
  DRIVER=$(podman inspect --format '{{.HostConfig.LogConfig.Type}}' "$c" 2>/dev/null)

  if [ -n "$LOG_PATH" ] && [ -f "$LOG_PATH" ]; then
    FILE_COUNT=$(ls "${LOG_PATH}"* 2>/dev/null | wc -l)
    TOTAL_SIZE=$(du -ch "${LOG_PATH}"* 2>/dev/null | tail -1 | awk '{print $1}')
    CURRENT_SIZE=$(du -h "$LOG_PATH" | awk '{print $1}')

    echo "$c ($DRIVER):"
    echo "  Files: $FILE_COUNT"
    echo "  Current: $CURRENT_SIZE"
    echo "  Total: $TOTAL_SIZE"
  fi
done
```

## Rotation in Podman Compose

```yaml
# podman-compose.yml
services:
  web:
    image: nginx:latest
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  api:
    image: my-api:latest
    logging:
      driver: k8s-file
      options:
        max-size: "50m"
        max-file: "5"

  database:
    image: postgres:16
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "3"
```

## Sizing Guidelines

```bash
# Quick reference for rotation settings:
#
# Development:
#   max-size: 5m, max-file: 2 (10MB total per container)
#
# Staging:
#   max-size: 20m, max-file: 3 (60MB total per container)
#
# Production (low volume):
#   max-size: 50m, max-file: 3 (150MB total per container)
#
# Production (high volume):
#   max-size: 100m, max-file: 5 (500MB total per container)
#
# Total disk budget = max_size * max_file * number_of_containers
# Example: 10 containers * 50m * 3 = 1.5GB for logs
```

## Summary

Log rotation in Podman is configured with `--log-opt max-size` and `--log-opt max-file`. These options work with both k8s-file and json-file log drivers. Set them per container at creation time, or configure system-wide defaults in `containers.conf`. Always calculate your total disk budget as `max-size * max-file * container-count` and monitor rotation health to ensure it is working as expected.
