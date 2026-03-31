# How to Export Container Configuration as Docker Run Command

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Container Configuration, Export, DevOps, Automation

Description: Export a running container's full configuration as a reproducible docker run command for documentation, migration, or converting container configs to Compose stacks.

---

This guide shows you how to accomplish this common container management task in Portainer, including both the UI approach and the equivalent command-line method.

## Using the Portainer UI

Navigate to **Containers** in the left sidebar. The container list view provides search and filter options at the top of the page.

### Filtering Options

The Portainer container list supports filtering by:
- **Status**: Running, Stopped, Exited, Paused
- **Name**: Search by container name substring
- **Stack**: Filter by stack name
- **Labels**: Filter by container labels (available in container details)

Click the **Filter** button or search box to apply your criteria. The container list updates in real time.

## Using the Docker CLI

For scripted or automated use cases, Docker CLI provides powerful filtering:

```bash
# Filter running containers

docker ps --filter "status=running"

# Filter by label key=value
docker ps --filter "label=com.docker.compose.service=webapp"

# Filter by stack name (using Compose label)
docker ps --filter "label=com.docker.compose.project=my-stack"

# Filter by image name
docker ps --filter "ancestor=nginx:1.25"

# Multiple filters (AND condition)
docker ps --filter "status=running" --filter "label=environment=production"

# Format the output to show specific fields
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Labels}}"
```

## Labeling Your Containers for Better Filtering

Apply meaningful labels to make filtering effective:

```yaml
# In your docker-compose.yml
services:
  webapp:
    image: myapp:1.2.3
    labels:
      # Standard labels for filtering
      environment: "production"
      team: "backend"
      tier: "api"
      version: "1.2.3"
      backup: "required"
```

With these labels, you can filter by:
```bash
# Find all production containers
docker ps --filter "label=environment=production"

# Find all containers owned by the backend team
docker ps --filter "label=team=backend"

# Find containers needing backup
docker ps --filter "label=backup=required"
```

## Using the Portainer API

For integration with monitoring tools or dashboards:

```python
import requests

headers = {"X-API-Key": "your-api-token"}

# Get containers with specific label via Portainer API
response = requests.get(
    "https://portainer.example.com/api/endpoints/1/docker/containers/json",
    headers=headers,
    params={
        "all": "false",  # Only running containers
        "filters": '{"label": ["environment=production"]}'
    }
)

containers = response.json()
for c in containers:
    print(c["Names"][0], c["Status"])
```

## Summary

Portainer's container filtering UI and Docker's filter flags both support filtering by status and labels. Consistent label conventions across your stacks make it significantly easier to find, manage, and audit specific container groups. Use the Portainer API for programmatic filtering in automation and monitoring tools.
