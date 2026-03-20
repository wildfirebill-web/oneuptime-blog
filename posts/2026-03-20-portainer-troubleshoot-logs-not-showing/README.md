# How to Troubleshoot Container Logs Not Showing in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Logging, Troubleshooting

Description: Learn how to diagnose and fix issues where Docker container logs are not appearing or showing empty in Portainer's log viewer.

## Introduction

One of the most frustrating Portainer issues is opening the log viewer and seeing nothing — blank logs for a running container. This can be caused by the wrong logging driver, application writing to files instead of stdout, output buffering, or Portainer's log viewer limitations. This guide walks through each cause systematically.

## Prerequisites

- Portainer installed with a connected Docker environment
- A container with empty or missing logs

## Step 1: Check the Logging Driver

Portainer's log viewer only supports `json-file` and `journald` drivers. Other drivers send logs elsewhere.

```bash
# Check which logging driver the container uses:
docker inspect my-container | jq '.[].HostConfig.LogConfig'

# Output if correct:
{
  "Type": "json-file",
  "Config": {}
}

# Output if problematic (logs go to Fluentd, not Portainer):
{
  "Type": "fluentd",
  "Config": {
    "fluentd-address": "localhost:24224"
  }
}
```

**Fix:** If using a non-json-file driver, switch to `json-file` (or `journald`) to see logs in Portainer:

```yaml
# docker-compose.yml: force json-file logging
services:
  app:
    image: myorg/app:latest
    logging:
      driver: json-file    # Portainer can show these logs
      options:
        max-size: "10m"
        max-file: "5"
```

Or set system-wide default:

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```

## Step 2: Verify the Application Logs to stdout

Docker captures stdout and stderr. If your application writes to a file, Portainer won't see it.

```bash
# Check if the container has any output at all:
docker logs my-container

# If empty, the app isn't writing to stdout/stderr

# Check if the app is writing to files:
docker exec my-container find /var/log -name "*.log" -size +0c 2>/dev/null
docker exec my-container ls -la /app/logs/ 2>/dev/null
```

**Fix:** Redirect log files to stdout/stderr in your Dockerfile or entrypoint:

```dockerfile
# Dockerfile: symlink log files to stdout
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log
```

Or in your application:

```python
# Python: ensure logging goes to stdout, not a file
import logging
import sys

logging.basicConfig(
    stream=sys.stdout,    # stdout — not a file
    level=logging.INFO,
    format='%(asctime)s %(levelname)s: %(message)s',
    force=True  # Override any existing handlers
)
```

```yaml
# Nginx: configure logging to stdout/stderr
# nginx.conf
http {
    # Log to stdout (captured by Docker)
    access_log /dev/stdout main;
    error_log /dev/stderr warn;
}
```

## Step 3: Disable Output Buffering

Some runtimes buffer output, which delays log appearance in Portainer.

### Python

```yaml
services:
  app:
    image: python:3.12-slim
    environment:
      # Disable Python output buffering
      - PYTHONUNBUFFERED=1
      # Also: PYTHONDONTWRITEBYTECODE=1
```

Or in code:

```python
import sys
# Flush after every write:
sys.stdout.reconfigure(line_buffering=True)
# Or:
import functools
print = functools.partial(print, flush=True)
```

### Node.js

Node.js doesn't buffer stdout by default, but in some cases:

```javascript
// Force immediate output:
process.stdout.write('message\n');

// Configure stream:
process.stdout.setDefaultEncoding('utf-8');
```

### PHP

```yaml
services:
  php:
    image: php:8.2-fpm
    environment:
      # Disable PHP output buffering
      - PHP_OUTPUT_BUFFERING=0
```

Or in `php.ini`:

```ini
output_buffering = Off
```

### Ruby

```yaml
services:
  ruby:
    image: ruby:3.3
    environment:
      # Disable Ruby output buffering
      - RUBY_STDOUT_SYNC=1
```

Or in code:

```ruby
$stdout.sync = true
$stderr.sync = true
```

## Step 4: Check Log Rotation (Old Logs Deleted)

If log rotation ran, old logs may be gone. This isn't a Portainer issue but can seem like logs are missing.

```bash
# Check if the json-file log exists:
CONTAINER_ID=$(docker inspect my-container --format '{{.Id}}')
sudo ls -la "/var/lib/docker/containers/${CONTAINER_ID}/"
# Look for: <id>-json.log

# Check log file size:
sudo du -sh "/var/lib/docker/containers/${CONTAINER_ID}/${CONTAINER_ID}-json.log"
```

If the file is empty or tiny, logs were either rotated or never written.

## Step 5: Verify Container Is Actually Running

```bash
# Check container is running:
docker ps | grep my-container

# Check exit code (if container keeps restarting):
docker inspect my-container | jq '.[].State.ExitCode'

# Check restart count:
docker inspect my-container | jq '.[].RestartCount'
```

If the container is restarting rapidly, check its logs immediately after start.

## Step 6: Check Log Size Limits

If logs are rotated and you're looking at lines from the current file only:

```bash
# Check actual log file size:
LOGPATH=$(docker inspect my-container --format '{{.LogPath}}')
sudo wc -l "${LOGPATH}"
sudo du -sh "${LOGPATH}"

# In Portainer log viewer:
# Change the number of lines from "100" to "All"
# to see all available logs
```

## Step 7: Portainer Agent Issues (Remote Hosts)

For remote environments, the Portainer Agent may have connection issues.

```bash
# Check agent status on the remote host:
docker ps | grep portainer_agent
docker logs portainer_agent --tail 20

# Verify agent can reach the Docker socket:
docker exec portainer_agent ls /var/run/docker.sock
```

## Step 8: Test Logging with a Known-Good Container

Rule out Portainer issues by testing with a simple container:

```bash
# Deploy a test container that definitely logs:
docker run -d \
  --name log-test \
  --restart no \
  alpine:latest \
  sh -c "while true; do echo 'Test log line: $(date)'; sleep 5; done"

# Check if Portainer shows logs for this container
# If yes: the issue is with your application's logging configuration
# If no: the issue is with Portainer or Docker logging infrastructure
```

## Summary Diagnostic Flow

```
Logs empty in Portainer?
    │
    ├─ Check logging driver: json-file? ─ No → Switch to json-file
    │
    ├─ Check docker logs <container>: empty? ─ Yes → App not writing to stdout
    │   └─ Fix: redirect app logs to stdout; disable buffering
    │
    ├─ Check docker logs <container>: has output? ─ Yes
    │   └─ This is a Portainer UI issue; check line count setting; refresh page
    │
    └─ Container kept restarting?
        └─ Logs from previous run may be gone; increase max-file for json-file driver
```

## Conclusion

Container logs not showing in Portainer is almost always due to one of three causes: the wrong logging driver (use `json-file`), the application writing to files instead of stdout, or output buffering preventing immediate flushing. Check the logging driver first, then verify stdout logging in your application, and enable unbuffered output for your runtime. Once logs flow to stdout with the `json-file` driver, Portainer will display them reliably.
