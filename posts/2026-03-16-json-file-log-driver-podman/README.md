# How to Use the json-file Log Driver with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Logging, JSON

Description: Learn how to configure and use the json-file log driver in Podman for Docker-compatible JSON log output, including log rotation and programmatic parsing.

---

> The json-file log driver stores each log line as a JSON object with a timestamp and stream identifier, making it easy to parse programmatically.

The json-file log driver writes container logs to a file in JSON format, with each line containing the log message, timestamp, and stream (stdout or stderr). This format is compatible with Docker's default log driver, making it useful for environments migrating from Docker to Podman.

---

## Enable the json-file Log Driver

```bash
# Run a container with the json-file log driver

podman run -d \
  --log-driver json-file \
  --name web \
  nginx:latest

# Verify the log driver
podman inspect --format '{{.HostConfig.LogConfig.Type}}' web

# View logs normally (podman logs still works)
podman logs web
```

## Understand the JSON Log Format

```bash
# Find the log file location
podman inspect --format '{{.LogPath}}' web

# View the raw JSON log file
cat $(podman inspect --format '{{.LogPath}}' web) | head -5

# Each line is a JSON object like:
# {"log":"172.17.0.1 - - [16/Mar/2026:14:30:01 +0000] \"GET / HTTP/1.1\" 200 615\n","stream":"stdout","time":"2026-03-16T14:30:01.123456789Z"}

# Pretty-print a log entry
cat $(podman inspect --format '{{.LogPath}}' web) | head -1 | python3 -m json.tool
```

The JSON structure contains:
- **log**: The actual log message (including trailing newline).
- **stream**: Either "stdout" or "stderr".
- **time**: RFC 3339 timestamp with nanosecond precision.

## Configure Log Rotation

Prevent log files from growing indefinitely with rotation options.

```bash
# Set maximum log file size
podman run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --name web \
  nginx:latest

# Note: In Podman, json-file is aliased to the k8s-file driver.
# The max-size option is supported, but max-file (number of rotated files)
# is not currently supported by Podman's logging drivers.

# Check the current log file size
LOG_PATH=$(podman inspect --format '{{.LogPath}}' web)
ls -lh "$LOG_PATH"
```

## Parse JSON Logs Programmatically

The JSON format makes logs easy to process with standard tools.

```bash
# Extract just the log messages using jq
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r '.log'

# Filter by stream (stdout only)
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r 'select(.stream == "stdout") | .log'

# Filter by stream (stderr only)
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r 'select(.stream == "stderr") | .log'

# Extract timestamps and messages
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r '[.time, .stream, .log] | @tsv'

# Filter by time range
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r '
  select(.time > "2026-03-16T14:00:00" and .time < "2026-03-16T15:00:00") | .log
'

# Count entries by stream type
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r '.stream' | sort | uniq -c
```

## Search JSON Logs

```bash
# Search for errors in the JSON log messages
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r 'select(.log | test("error"; "i")) | .log'

# Search with context (timestamp + message)
cat $(podman inspect --format '{{.LogPath}}' web) | jq -r '
  select(.log | test("error"; "i")) | "\(.time) \(.log)"
'

# Count errors
cat $(podman inspect --format '{{.LogPath}}' web) | jq '[select(.log | test("error"; "i"))] | length'

# Search across rotated files
LOG_PATH=$(podman inspect --format '{{.LogPath}}' web)
cat "$LOG_PATH"* | jq -r 'select(.log | test("error"; "i")) | .log'
```

## Process Logs with Python

For more complex analysis, use Python to process JSON logs.

```bash
# Quick analysis script
LOG_PATH=$(podman inspect --format '{{.LogPath}}' web)

python3 << PYEOF
import json
import sys

errors = 0
warnings = 0
total = 0

with open("$LOG_PATH") as f:
    for line in f:
        entry = json.loads(line)
        total += 1
        msg = entry.get("log", "").lower()
        if "error" in msg:
            errors += 1
        if "warn" in msg:
            warnings += 1

print(f"Total entries: {total}")
print(f"Errors: {errors}")
print(f"Warnings: {warnings}")
PYEOF
```

## Set as Default Log Driver

```bash
# Configure json-file as the default for all new containers
mkdir -p ~/.config/containers

cat >> ~/.config/containers/containers.conf << 'EOF'
[containers]
log_driver = "json-file"
log_size_max = 10485760
EOF

# The log_size_max value is in bytes (10485760 = 10MB)
# Note: json-file is aliased to k8s-file in Podman

# Verify the configuration
podman info --format '{{.Host.LogDriver}}'
```

## Use in Podman Compose

```yaml
# podman-compose.yml
services:
  web:
    image: nginx:latest
    logging:
      driver: json-file
      options:
        max-size: "10m"

  api:
    image: my-api:latest
    logging:
      driver: json-file
      options:
        max-size: "50m"
```

## Summary

The json-file log driver stores container logs as structured JSON, making them easy to parse with tools like `jq` and Python. Configure the `max-size` option to limit log file size and prevent disk exhaustion. Note that in Podman, json-file is aliased to the k8s-file driver. The JSON format is Docker-compatible, making it a good choice for environments transitioning from Docker to Podman or for workflows that require programmatic log processing.
