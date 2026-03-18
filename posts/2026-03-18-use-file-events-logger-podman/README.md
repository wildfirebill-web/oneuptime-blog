# How to Use File Events Logger with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Monitoring, Events, Logging, File

Description: Learn how to use the file events logger with Podman to store container events in a plain text file for simple and portable event logging.

---

> The file events logger provides straightforward, portable event storage without any dependency on systemd or external services.

Not every system runs systemd, and not every use case requires the complexity of journald. Podman's file events logger stores container events in a simple log file that can be read, parsed, and managed with standard Unix tools. This guide shows you how to configure and use the file events logger effectively.

---

## Enabling the File Backend

Configure Podman to use the file events logger.

```bash
# Create the configuration directory
mkdir -p ~/.config/containers/

# Set file as the events logger
cat > ~/.config/containers/containers.conf << 'EOF'
[engine]
events_logger = "file"
EOF

# Verify the change
podman info --format '{{.Host.EventLogger}}'
```

## Understanding the Events File Location

The file backend writes events to a specific file path.

```bash
# Check where events are stored
podman info --format '{{.Store.EventsLogFilePath}}'

# The default location for rootless Podman is typically:
# ~/.local/share/containers/storage/events/events.log

# For rootful Podman:
# /var/lib/containers/storage/events/events.log

# Check if the file exists and its size
EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')
ls -lh "$EVENTS_FILE" 2>/dev/null || echo "Events file does not exist yet"
```

## Generating Test Events

Create some events to populate the events file.

```bash
# Generate various container events
podman run --rm --name file-test-1 alpine echo "hello from file logger"
podman run --rm --name file-test-2 alpine echo "second test"
podman run -d --name file-test-3 alpine sleep 300
podman stop file-test-3
podman rm file-test-3
```

## Reading the Events File Directly

Since events are stored in a plain text file, you can read them with standard tools.

```bash
# Get the events file path
EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')

# View the raw events file
cat "$EVENTS_FILE"

# View the last 20 events
tail -20 "$EVENTS_FILE"

# Follow the file for new events in real-time
tail -f "$EVENTS_FILE"
```

## Querying Events Through Podman

The `podman events` command works the same regardless of backend.

```bash
# View recent events
podman events --since 10m

# Filter by event type
podman events --since 10m --filter event=start

# Filter by container
podman events --since 10m --filter container=file-test-1

# JSON output
podman events --since 10m --format json
```

## Configuring Events File Size

The file backend supports a maximum log file size to prevent the file from growing indefinitely.

```bash
# Set maximum events log file size in containers.conf
cat > ~/.config/containers/containers.conf << 'EOF'
[engine]
events_logger = "file"

# Maximum size in bytes before rotation
# Default is 1MB (1000000 bytes)
# Set to 5MB
events_log_file_size = 5000000
EOF

# Verify the setting
podman info --format '{{.Store.EventsLogFileSize}}'
```

## Parsing the Events File

The file format is structured and can be parsed with standard text tools.

```bash
# Get the events file
EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')

# Count events by type
awk '{print $3}' "$EVENTS_FILE" | sort | uniq -c | sort -rn

# Find all die events
grep " die " "$EVENTS_FILE"

# Extract timestamps of start events
grep " start " "$EVENTS_FILE" | awk '{print $1}'
```

## Managing the Events File

Unlike journald, the file backend requires manual management.

```bash
#!/bin/bash
# manage-events-file.sh - Manage the Podman events log file

EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')

echo "Events file: ${EVENTS_FILE}"

if [ -f "$EVENTS_FILE" ]; then
    echo "File size: $(du -h "$EVENTS_FILE" | cut -f1)"
    echo "Total events: $(wc -l < "$EVENTS_FILE")"
    echo "First event: $(head -1 "$EVENTS_FILE")"
    echo "Last event: $(tail -1 "$EVENTS_FILE")"
else
    echo "Events file does not exist"
fi
```

## Setting Up Log Rotation

For long-running systems, set up log rotation to manage the events file.

```bash
# Create a logrotate configuration for Podman events
# For rootless: this goes in a user cron job or timer
EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')

cat > /tmp/podman-events-logrotate.conf << EOF
${EVENTS_FILE} {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    copytruncate
}
EOF

# Test the logrotate configuration
logrotate -d /tmp/podman-events-logrotate.conf
```

## Building a File-Based Event Monitor

```bash
#!/bin/bash
# file-event-monitor.sh - Monitor Podman events via the file backend

EVENTS_FILE=$(podman info --format '{{.Store.EventsLogFilePath}}')

if [ ! -f "$EVENTS_FILE" ]; then
    echo "Events file not found. Generating a test event..."
    podman run --rm alpine echo "init" > /dev/null 2>&1
fi

echo "Monitoring Podman events file: ${EVENTS_FILE}"
echo "Press Ctrl+C to stop"
echo "---"

# Use tail -f to follow new events
tail -f "$EVENTS_FILE" | while IFS= read -r line; do
    # Parse the event line
    timestamp=$(echo "$line" | awk '{print $1}')
    event_type=$(echo "$line" | awk '{print $3}')

    # Highlight critical events
    case "$event_type" in
        die|oom)
            echo "CRITICAL: $line"
            ;;
        stop|kill)
            echo "WARNING:  $line"
            ;;
        *)
            echo "INFO:     $line"
            ;;
    esac
done
```

## Advantages of the File Backend

The file backend is best suited for specific scenarios:

```bash
# Check if systemd is available on your system
# If not, file backend is your best option
systemctl --version 2>/dev/null || echo "systemd not available - use file backend"

# The file backend works well for:
# - Systems without systemd (Alpine, containers, minimal distros)
# - Simple setups where text file parsing is sufficient
# - Environments where journal access is restricted
# - Testing and development environments
```

## Cleanup

```bash
# Clean up test containers
podman rm -f file-test-3 2>/dev/null
```

## Summary

The file events logger in Podman provides a simple, dependency-free way to store container events. Events are written to a plain text file that can be read with standard Unix tools, managed with logrotate, and parsed with grep, awk, or any text processing tool. While it lacks the structured querying capabilities of journald, the file backend is portable, straightforward, and works on any system regardless of init system. It is an excellent choice for minimal environments, containers, and simple monitoring setups.
