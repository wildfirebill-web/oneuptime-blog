# How to Duplicate a Container in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Operations, DevOps

Description: Learn how to duplicate an existing Docker container in Portainer to create a copy with the same configuration for scaling or testing purposes.

## Introduction

Portainer's duplicate container feature creates a new container pre-populated with the same settings as an existing one. This is useful for scaling services, creating test copies of production containers, or applying incremental configuration changes without starting from scratch.

## Prerequisites

- Portainer installed with a connected Docker environment
- An existing container to duplicate

## Step 1: Find the Duplicate Option

1. Navigate to **Containers** in Portainer.
2. Click on the container name you want to duplicate.
3. On the container details page, look for the **Duplicate/Edit** button.

In some Portainer versions:
- The button appears as **Duplicate/Edit** on the container details page.
- Or a duplicate icon in the container list actions.

## Step 2: Review the Pre-Populated Form

Clicking **Duplicate/Edit** opens the container creation form pre-filled with all the original container's settings:

- Image name and tag
- Port mappings
- Volume mounts
- Environment variables
- Labels
- Restart policy
- Resource limits
- Networking configuration
- Command and entrypoint

## Step 3: Modify the Duplicate

Before deploying, change the settings that need to differ from the original:

### Change the Container Name

The most important change — you can't have two containers with the same name:

```
Original name:   web-server
Duplicate name:  web-server-2
```

### Adjust Port Mappings

Change host ports to avoid conflicts:

```
Original:   8080 → 80
Duplicate:  8081 → 80
```

### Modify Environment Variables

For copies with different configurations:

```
Original:   INSTANCE_ID=1, WORKER_TYPE=primary
Duplicate:  INSTANCE_ID=2, WORKER_TYPE=secondary
```

### Change Labels

Update labels to reflect the new instance:

```
Original:   instance=1, role=primary
Duplicate:  instance=2, role=secondary
```

## Step 4: Deploy the Duplicate

After modifying the necessary settings:
1. Review all configurations.
2. Click **Deploy the container**.

The new container starts alongside the original.

## Common Use Cases

### Scaling a Web Service

Quickly add another instance of a web server:

```
Container: web-app (port 8080→80)
Duplicate: web-app-2 (port 8081→80)

# Then point a load balancer at both ports
```

### Testing Configuration Changes

Create a copy to test changes without touching the running original:

```
Container: production-api (env: LOG_LEVEL=warn)
Duplicate: production-api-test (env: LOG_LEVEL=debug)
```

Test the duplicate, and if everything works, apply the changes to the original.

### Creating Dev Copy of Production Container

```
Container: prod-postgres (data volume: prod-data)
Duplicate: dev-postgres (data volume: dev-data, password: devpassword)
```

Change only the volume mount and password — keep everything else identical.

## Using Docker Compose for Proper Scaling

While duplicating containers manually works for quick tasks, for production scaling, use Docker Compose replicas:

```yaml
# docker-compose.yml with scaling
version: "3.8"

services:
  web:
    image: myorg/webapp:latest
    restart: unless-stopped
    deploy:
      replicas: 3   # Run 3 copies automatically
    ports:
      - "8080-8082:8080"  # Map a range of host ports
```

Or with a load balancer:

```yaml
services:
  web:
    image: myorg/webapp:latest
    deploy:
      replicas: 3
    # No direct host port mapping — nginx routes to all instances

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
```

Nginx upstream configuration:

```nginx
# nginx.conf for load balancing across duplicated containers
upstream web_backend {
    server web:8080;  # Docker DNS resolves to all container IPs
}
```

## Limitations of Manual Duplication

- **No automatic port assignment**: You must manually change ports to avoid conflicts.
- **No orchestration**: Duplicated containers aren't managed as a group.
- **No auto-scaling**: Each duplicate is individually managed.
- **Shared volumes**: Be careful — duplicated containers may share the same volume unless you change the volume configuration.

For production scaling needs, use Docker Swarm services or Kubernetes instead.

## Duplicating a Container vs. Recreating It

| Action | Purpose |
|--------|---------|
| **Duplicate** | Create a second copy alongside the original |
| **Edit (same name)** | Update settings; must remove old container first |
| **Recreate** | Remove old container, create new one with updated config |

## Step 5: Track Duplicated Containers

Use labels to track which containers are duplicates and of what:

```
Labels:
  com.example.cloned-from: original-container-name
  com.example.instance: 2
  com.example.created-by: john-doe
```

This makes it easy to identify and manage duplicated containers in large environments.

## Conclusion

Duplicating containers in Portainer is a quick way to create copies of existing containers for scaling, testing, or creating dev copies of production configs. It's most useful for ad-hoc operations where you need a second instance quickly. For systematic scaling and production workloads, use Docker Compose replicas or Docker Swarm services to manage multiple instances as a unit rather than individually.
