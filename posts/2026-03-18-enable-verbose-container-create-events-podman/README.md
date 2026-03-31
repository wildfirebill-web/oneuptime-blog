# How to Enable Verbose Container Create Events in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Monitoring, Event, Debugging, Verbose

Description: Learn how to enable verbose container create events in Podman to capture detailed information about container configuration at creation time.

---

> Verbose create events capture the full story of how a container was configured, turning event logs into a complete audit trail.

By default, Podman create events contain only basic information about the container. Enabling verbose mode adds detailed configuration data to create events, including the full command line, environment variables, mount points, and other settings. This extra detail is invaluable for auditing, debugging, and understanding exactly how containers were configured. This guide shows you how to enable and use verbose container create events.

---

## Understanding Default Create Events

First, observe what a standard create event looks like.

```bash
# Generate a create event

podman run --rm --name verbose-test alpine echo "hello"

# View the recent create event
podman events --since 1m --filter event=create
```

The default create event shows the container name, ID, image, and timestamp but not the full configuration.

## Enabling Verbose Events

Verbose events are configured in the `containers.conf` file.

```bash
# Create the configuration directory
mkdir -p ~/.config/containers/

# Enable verbose events
cat >> ~/.config/containers/containers.conf << 'EOF'
[engine]
# Enable verbose create events to capture full container configuration
events_container_create_inspect_data = true
EOF

# Verify Podman reads the configuration
podman info --format '{{.Host.EventLogger}}'
```

## Viewing Verbose Create Events

With verbose mode enabled, create events contain much more detail.

```bash
# Create a container with various options
podman run -d --name verbose-demo \
    -e APP_ENV=production \
    -e DB_HOST=localhost \
    -p 8080:80 \
    -v /tmp/data:/data:ro \
    --memory 256m \
    --cpus 0.5 \
    --label app=webserver \
    --label version=1.0 \
    alpine sleep 300

# View the verbose create event in JSON format
podman events --since 1m --filter event=create --format json | jq '.'
```

## Extracting Configuration from Verbose Events

The verbose event data includes the container inspection data at creation time.

```bash
# Get the full create event with inspection data
podman events --since 5m --filter event=create --format json | \
    jq 'select(.Actor.Attributes.name == "verbose-demo")'

# Extract specific configuration details
podman events --since 5m --filter event=create --format json | \
    jq 'select(.Actor.Attributes.name == "verbose-demo") | .Actor.Attributes'
```

## Using Verbose Events for Auditing

Verbose create events serve as an audit log of container configurations.

```bash
#!/bin/bash
# audit-creates.sh - Audit container creations using verbose events

echo "=== Container Creation Audit Report ==="
echo "Generated: $(date)"
echo ""

# Get all create events from the last 24 hours
podman events --since 24h --filter event=create --format json | \
while IFS= read -r event; do
    name=$(echo "$event" | jq -r '.Actor.Attributes.name // "unnamed"')
    image=$(echo "$event" | jq -r '.Actor.Attributes.image // "unknown"')
    timestamp=$(echo "$event" | jq -r '.time')

    echo "Container: ${name}"
    echo "  Image: ${image}"
    echo "  Created: ${timestamp}"

    # Extract labels if available in verbose data
    labels=$(echo "$event" | jq -r '.Actor.Attributes | to_entries[] | select(.key | startswith("label_")) | "  Label: \(.key | sub("label_";"")) = \(.value)"' 2>/dev/null)
    if [ -n "$labels" ]; then
        echo "$labels"
    fi

    echo ""
done
```

## Comparing Verbose and Non-Verbose Events

See the difference in event detail level.

```bash
# Create a container to compare event formats
podman create --name compare-test \
    -e SECRET=mysecret \
    -p 9090:80 \
    --memory 128m \
    alpine sleep 60

# View the event with standard format
podman events --since 1m --filter event=create \
    --format '{{.Time}} {{.Status}} {{.Name}}'

# View the same event with full JSON (verbose data included)
podman events --since 1m --filter event=create --format json | \
    jq 'select(.Actor.Attributes.name == "compare-test") | keys'

# Clean up
podman rm compare-test
```

## Capturing Verbose Events to a Database

Store verbose event data for long-term analysis.

```bash
#!/bin/bash
# store-verbose-events.sh - Store verbose create events in a SQLite database

DB_FILE="/tmp/podman-audit.db"

# Create the database table
sqlite3 "$DB_FILE" << 'SQL'
CREATE TABLE IF NOT EXISTS container_creates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT,
    name TEXT,
    image TEXT,
    event_data TEXT,
    created_at TEXT DEFAULT (datetime('now'))
);
SQL

echo "Monitoring verbose create events..."
echo "Database: ${DB_FILE}"

# Stream create events and store in database
podman events --filter event=create --format json | \
while IFS= read -r event; do
    name=$(echo "$event" | jq -r '.Actor.Attributes.name // "unnamed"')
    image=$(echo "$event" | jq -r '.Actor.Attributes.image // "unknown"')
    timestamp=$(echo "$event" | jq -r '.time')

    # Escape single quotes for SQL
    safe_event=$(echo "$event" | sed "s/'/''/g")

    sqlite3 "$DB_FILE" \
        "INSERT INTO container_creates (timestamp, name, image, event_data) VALUES ('${timestamp}', '${name}', '${image}', '${safe_event}');"

    echo "Stored: ${name} (${image})"
done
```

## Security Considerations

Verbose create events may contain sensitive information.

```bash
# Verbose events may include environment variables
# Be careful with sensitive data in container configs

# Good practice: use secrets or files instead of env vars
podman run --rm \
    --secret db-password \
    --name secure-container \
    alpine cat /run/secrets/db-password

# If you must use env vars, be aware they appear in verbose events
# Restrict access to the events log file or journal
chmod 600 "$(podman info --format '{{.Store.EventsLogFilePath}}')" 2>/dev/null
```

## Disabling Verbose Events

If you need to turn off verbose events:

```bash
# Edit containers.conf to disable verbose events
# Set events_container_create_inspect_data = false
# or remove the line entirely

# Verify the change
podman info --format '{{.Host.EventLogger}}'
```

## Cleanup

```bash
# Remove test containers
podman rm -f verbose-demo compare-test 2>/dev/null

# Remove test database
rm -f /tmp/podman-audit.db
```

## Summary

Enabling verbose container create events in Podman enriches your event logs with full container configuration details captured at creation time. This provides a complete audit trail of how containers were set up, including environment variables, port mappings, resource limits, and labels. While powerful for auditing and debugging, be mindful that verbose events may contain sensitive data that should be protected. By combining verbose events with structured storage, you can build comprehensive container auditing systems.
