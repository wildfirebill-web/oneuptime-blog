# How to Run a Container with Labels in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Labels, Metadata

Description: Learn how to attach metadata labels to Podman containers for organization, filtering, automation, and monitoring integration.

---

> Labels turn containers from anonymous processes into organized, queryable, and manageable units of your infrastructure.

Labels are key-value pairs attached to containers as metadata. They do not affect the container's runtime behavior, but they are essential for organizing, filtering, and automating container management. Labels let you categorize containers by application, environment, owner, version, or any other dimension that matters to your workflow.

---

## Adding Labels to a Container

Use the `--label` (or `-l`) flag to attach labels:

```bash
# Add a single label

podman run -d --name web \
  --label app=frontend \
  nginx:latest

# Add multiple labels
podman run -d --name api \
  --label app=backend \
  --label env=production \
  --label team=platform \
  --label version=2.1.0 \
  alpine sleep infinity
```

## Using a Label File

For many labels, use `--label-file` to read labels from a file:

```bash
# Create a label file
cat > /tmp/labels.txt << 'EOF'
app=myapp
env=staging
team=engineering
version=1.5.0
maintainer=ops@example.com
cost-center=engineering-42
EOF

# Run a container with labels from the file
podman run -d --name labeled-app \
  --label-file /tmp/labels.txt \
  alpine sleep infinity

# Verify the labels
podman inspect labeled-app --format '{{json .Config.Labels}}' | python3 -m json.tool
```

## Inspecting Container Labels

```bash
# View all labels on a container
podman inspect api --format '{{json .Config.Labels}}' | python3 -m json.tool

# Get a specific label value
podman inspect api --format '{{index .Config.Labels "app"}}'

# List containers with their labels
podman ps --format "table {{.Names}}\t{{.Labels}}"
```

## Filtering Containers by Label

Labels are most useful when filtering:

```bash
# Create several labeled containers
podman run -d --name web-prod --label env=production --label app=web alpine sleep infinity
podman run -d --name web-staging --label env=staging --label app=web alpine sleep infinity
podman run -d --name api-prod --label env=production --label app=api alpine sleep infinity
podman run -d --name api-staging --label env=staging --label app=api alpine sleep infinity

# Filter by a single label
podman ps --filter label=env=production

# Filter by label existence (any value)
podman ps --filter label=env

# Filter by multiple labels (AND logic)
podman ps --filter label=env=production --filter label=app=web

# Use labels with podman stop/rm
podman stop $(podman ps -q --filter label=env=staging)
```

## Common Labeling Conventions

```bash
# Reverse-DNS style labels (recommended for namespacing)
podman run -d --name app \
  --label com.example.app.name=myservice \
  --label com.example.app.version=3.0.1 \
  --label com.example.app.component=api \
  --label com.example.app.managed-by=podman \
  alpine sleep infinity

# OCI standard labels
podman run -d --name oci-labeled \
  --label org.opencontainers.image.title="My Application" \
  --label org.opencontainers.image.version="1.0.0" \
  --label org.opencontainers.image.vendor="My Company" \
  --label org.opencontainers.image.source="https://github.com/example/app" \
  alpine sleep infinity
```

## Labels for Monitoring and Logging

```bash
# Labels that monitoring tools can use for grouping
podman run -d --name monitored-app \
  --label prometheus.io/scrape=true \
  --label prometheus.io/port=9090 \
  --label prometheus.io/path=/metrics \
  --label logging.driver=json-file \
  --label logging.level=info \
  alpine sleep infinity

# Query containers that should be scraped by Prometheus
podman ps --filter label=prometheus.io/scrape=true --format "{{.Names}}"
```

## Labels for Automation Scripts

```bash
# Use labels to drive automation
# Tag containers with backup and cleanup policies
podman run -d --name db-container \
  --label backup=daily \
  --label retention=30d \
  --label cleanup=never \
  alpine sleep infinity

podman run -d --name temp-worker \
  --label backup=none \
  --label cleanup=after-completion \
  --label ttl=1h \
  alpine sleep infinity

# Automation script to clean up containers tagged for cleanup
echo "Containers marked for cleanup:"
podman ps -a --filter label=cleanup=after-completion --format "{{.Names}}"

# Find containers with specific backup policies
echo "Containers needing daily backup:"
podman ps --filter label=backup=daily --format "{{.Names}}"
```

## Labels vs Environment Variables

Labels and environment variables serve different purposes:

```bash
# Labels: metadata for the container management layer
# Environment variables: configuration for the application inside the container
podman run -d --name proper-config \
  --label app=myservice \
  --label env=production \
  --label version=2.0 \
  -e DATABASE_URL=postgres://db:5432/mydb \
  -e LOG_LEVEL=info \
  -e PORT=8080 \
  alpine sleep infinity

# Labels are visible via inspect, not inside the container
podman inspect proper-config --format '{{index .Config.Labels "app"}}'

# Environment variables are visible inside the container
podman exec proper-config sh -c 'echo $DATABASE_URL'
```

## Bulk Operations Using Labels

```bash
# Stop all production containers
podman stop $(podman ps -q --filter label=env=production) 2>/dev/null

# Remove all staging containers
podman rm -f $(podman ps -aq --filter label=env=staging) 2>/dev/null

# List all unique label values for a given key
podman ps -a --format '{{.Labels}}' | tr ',' '\n' | grep "env=" | sort -u
```

## Summary

Labels are a simple but powerful mechanism for container organization:

- Use `--label key=value` to attach metadata to containers
- Use `--label-file` for multiple labels from a file
- Filter containers with `--filter label=key=value`
- Follow reverse-DNS naming for label namespacing
- Use labels for monitoring integration, automation, and resource grouping
- Labels are metadata only and do not affect container runtime behavior

A consistent labeling strategy across your containers makes management, monitoring, and automation significantly easier.
