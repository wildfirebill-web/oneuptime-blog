# How to Filter Container List by Label in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Container Management, Filtering, Labels

Description: Learn how to use labels to organize containers and filter Podman listings by label key-value pairs for efficient management.

---

> Labels provide a flexible metadata system for organizing containers, enabling powerful filtering and management across large deployments.

Labels are key-value pairs attached to containers that serve as metadata for organization, filtering, and automation. Podman lets you filter container listings by label, making it easy to find containers belonging to specific projects, environments, or services. This guide covers how to use labels effectively.

---

## Adding Labels to Containers

Attach labels when creating or running containers using the `--label` (or `-l`) flag.

```bash
# Run containers with labels
podman run -d --name web-prod \
  --label app=mywebsite \
  --label environment=production \
  --label team=frontend \
  docker.io/library/nginx:latest

podman run -d --name web-staging \
  --label app=mywebsite \
  --label environment=staging \
  --label team=frontend \
  docker.io/library/nginx:latest

podman run -d --name api-prod \
  --label app=myapi \
  --label environment=production \
  --label team=backend \
  docker.io/library/alpine:latest sleep 300

podman run -d --name db-prod \
  --label app=mydb \
  --label environment=production \
  --label team=data \
  docker.io/library/alpine:latest sleep 300
```

## Filtering by Label Key

Filter for containers that have a specific label, regardless of its value.

```bash
# Find all containers with the "environment" label
podman ps -a --filter label=environment

# Find all containers with the "app" label
podman ps -a --filter label=app
```

## Filtering by Label Key and Value

```bash
# Find production containers
podman ps -a --filter label=environment=production

# Find staging containers
podman ps -a --filter label=environment=staging

# Find containers for a specific app
podman ps -a --filter label=app=mywebsite
```

## Combining Multiple Label Filters

Multiple label filters work as AND conditions.

```bash
# Find production containers for the frontend team
podman ps -a \
  --filter label=environment=production \
  --filter label=team=frontend

# Find production containers for a specific app
podman ps -a \
  --filter label=environment=production \
  --filter label=app=myapi
```

## Combining Label with Other Filters

```bash
# Running production containers
podman ps \
  --filter label=environment=production \
  --filter status=running

# Exited containers for a specific team
podman ps -a \
  --filter label=team=backend \
  --filter status=exited

# Production containers with a specific name pattern
podman ps -a \
  --filter label=environment=production \
  --filter name=web
```

## Custom Format with Label Filters

```bash
# Show filtered containers with custom columns
podman ps -a \
  --filter label=environment=production \
  --format "table {{.Names}}\t{{.Status}}\t{{.Labels}}"

# Show only names and specific label values
podman ps -a \
  --filter label=environment=production \
  --format "{{.Names}}"
```

## Viewing Container Labels

```bash
# Show all labels for a specific container
podman inspect web-prod --format '{{.Config.Labels}}'

# Show labels in a readable format
podman inspect web-prod --format '{{range $k, $v := .Config.Labels}}{{$k}}={{$v}}{{"\n"}}{{end}}'

# Show labels for all running containers
podman ps --format "{{.Names}}\t{{.Labels}}"
```

## Using Labels from a File

```bash
# Create a label file
cat > /tmp/labels.txt << 'EOF'
app=myservice
environment=production
version=2.1.0
team=platform
EOF

# Run a container with labels from the file
podman run -d --name labeled-app \
  --label-file /tmp/labels.txt \
  docker.io/library/alpine:latest sleep 300

# Verify the labels
podman inspect labeled-app --format '{{.Config.Labels}}'

# Clean up
podman rm -f labeled-app
```

## Practical Use Cases

### Environment-Based Management

```bash
#!/bin/bash
# env-manage.sh - Manage containers by environment

ACTION=$1  # start, stop, restart, remove
ENV=$2     # production, staging, development

if [ -z "$ACTION" ] || [ -z "$ENV" ]; then
  echo "Usage: $0 <start|stop|restart|remove> <environment>"
  exit 1
fi

CONTAINERS=$(podman ps -a --filter label=environment="$ENV" -q)

if [ -z "$CONTAINERS" ]; then
  echo "No containers found for environment: $ENV"
  exit 0
fi

echo "$ACTION containers in $ENV environment..."
echo "$CONTAINERS" | xargs -r podman "$ACTION"
echo "Done"
```

### Service Discovery by Label

```bash
# Find all containers providing a specific service
podman ps --filter label=service=http \
  --format "{{.Names}}\t{{.Ports}}"

# Get IP addresses of all web services
for name in $(podman ps --filter label=service=http --format "{{.Names}}"); do
  IP=$(podman inspect "$name" --format '{{.NetworkSettings.IPAddress}}')
  echo "$name: $IP"
done
```

### Cleanup by Label

```bash
# Remove all development containers
podman ps -a --filter label=environment=development -q | xargs -r podman rm -f

# Remove containers with a specific app label
podman ps -a --filter label=app=test-app -q | xargs -r podman rm -f

# Prune containers by label
podman container prune --force --filter label=temporary=true
```

### Reporting by Team

```bash
#!/bin/bash
# team-report.sh - Container report by team

TEAMS=("frontend" "backend" "data" "platform")

echo "Container Report by Team"
echo "========================"

for team in "${TEAMS[@]}"; do
  RUNNING=$(podman ps --filter label=team="$team" -q | wc -l)
  TOTAL=$(podman ps -a --filter label=team="$team" -q | wc -l)
  echo ""
  echo "Team: $team (Running: $RUNNING / Total: $TOTAL)"
  podman ps -a --filter label=team="$team" \
    --format "  {{.Names}}\t{{.Status}}"
done
```

## Summary

Labels provide a powerful metadata system for organizing and filtering containers in Podman. Use `--label` to attach metadata when creating containers, and `--filter label=` to query them. Combine label filters with status, name, and other filters for precise container management. Labels are especially valuable in multi-team, multi-environment setups where naming conventions alone are insufficient.
