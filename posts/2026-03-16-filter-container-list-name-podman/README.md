# How to Filter Container List by Name in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Container Management, Filtering

Description: Learn how to filter Podman container listings by name using exact matches, patterns, and regular expressions.

---

> Filtering containers by name lets you quickly find specific services in environments with dozens or hundreds of containers.

When your system runs many containers, finding the right one by scrolling through the full list is impractical. The `--filter name=` option lets you search containers by name. This guide covers all the ways to filter containers by name in Podman.

---

## Basic Name Filtering

The `--filter name=` option matches container names that contain the given string.

```bash
# Create some test containers

podman run -d --name web-frontend docker.io/library/alpine:latest sleep 300
podman run -d --name web-backend docker.io/library/alpine:latest sleep 300
podman run -d --name db-primary docker.io/library/alpine:latest sleep 300
podman run -d --name db-replica docker.io/library/alpine:latest sleep 300

# Filter by name containing "web"
podman ps --filter name=web

# Filter by name containing "db"
podman ps --filter name=db
```

## Name Filter Uses Substring Matching

The name filter matches any container whose name contains the given string.

```bash
# This matches "web-frontend" and "web-backend"
podman ps --filter name=web

# This matches "db-primary" and "db-replica"
podman ps --filter name=db

# This matches "web-frontend" only
podman ps --filter name=frontend

# This matches "web-backend" and "db-primary" (both contain "b")
podman ps --filter name=b
```

## Using Regular Expressions

The name filter supports regular expressions for more precise matching.

```bash
# Match names starting with "web"
podman ps -a --filter name=^web

# Match names ending with "primary"
podman ps -a --filter name=primary$

# Match exact name
podman ps -a --filter name=^web-frontend$

# Match names with a digit
podman ps -a --filter name=[0-9]
```

## Including Stopped Containers

By default, `podman ps` only shows running containers. Add `-a` to include all states.

```bash
# Filter all containers (including stopped) by name
podman ps -a --filter name=web

# Filter only running containers by name
podman ps --filter name=web
```

## Combining Name with Other Filters

```bash
# Running containers with "web" in the name
podman ps --filter name=web --filter status=running

# Exited containers with "db" in the name
podman ps -a --filter name=db --filter status=exited

# Containers with "api" in the name using a specific image
podman ps -a --filter name=api --filter ancestor=docker.io/library/node:20-alpine
```

## Custom Format with Name Filter

```bash
# Show filtered containers with custom columns
podman ps --filter name=web --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Get just the names
podman ps --filter name=web --format "{{.Names}}"

# Get IDs for further processing
podman ps --filter name=web -q
```

## Practical Use Cases

### Managing Service Groups

```bash
# List all frontend services
podman ps --filter name=frontend --format "table {{.Names}}\t{{.Status}}"

# Stop all backend services
podman ps --filter name=backend -q | xargs -r podman stop

# Restart all API containers
podman ps --filter name=api -q | xargs -r podman restart
```

### Finding Containers by Project

When using naming conventions like `project-service-instance`:

```bash
# Find all containers for project "myapp"
podman ps -a --filter name=^myapp

# Find all containers for a specific environment
podman ps -a --filter name=prod
podman ps -a --filter name=staging
```

### Health Check by Name Pattern

```bash
#!/bin/bash
# health-check.sh - Check health of containers by name pattern

PATTERN="$1"

if [ -z "$PATTERN" ]; then
  echo "Usage: $0 <name-pattern>"
  exit 1
fi

echo "Health check for containers matching '$PATTERN':"
echo ""

for name in $(podman ps -a --filter name="$PATTERN" --format "{{.Names}}"); do
  STATUS=$(podman inspect "$name" --format '{{.State.Status}}')
  EXIT_CODE=$(podman inspect "$name" --format '{{.State.ExitCode}}')

  if [ "$STATUS" = "running" ]; then
    echo "  $name: RUNNING"
  elif [ "$STATUS" = "exited" ] && [ "$EXIT_CODE" = "0" ]; then
    echo "  $name: EXITED (clean)"
  else
    echo "  $name: $STATUS (exit code: $EXIT_CODE)"
  fi
done
```

### Batch Operations by Name

```bash
# Remove all containers with "test" in the name
podman ps -a --filter name=test -q | xargs -r podman rm -f

# Get logs from all containers matching a pattern
for name in $(podman ps --filter name=worker --format "{{.Names}}"); do
  echo "=== $name ==="
  podman logs --tail 5 "$name"
  echo ""
done

# Inspect all containers matching a name
podman ps --filter name=web --format "{{.Names}}" | \
  xargs -I {} podman inspect {} --format '{{.Name}}: {{.NetworkSettings.IPAddress}}'
```

## Multiple Name Filters

Use multiple `--filter name=` options to match containers with any of the specified patterns.

```bash
# Match containers with "web" OR "api" in the name
podman ps -a --filter name=web --filter name=api
```

Note: Multiple name filters work as an OR condition.

## Summary

Filtering containers by name with `--filter name=` is essential for managing multi-container environments. The filter uses substring matching by default and supports regular expressions for precise queries. Combine with `-q` for scripting, `--format` for custom output, and other filters like `--filter status=` for targeted container management.
