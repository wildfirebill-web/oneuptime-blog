# How to Migrate from Docker CLI to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Migration, CLI, DevOps

Description: Learn how to transition from managing containers with the Docker CLI to using Portainer's web UI for improved visibility and team collaboration.

## Introduction

The Docker CLI is powerful but requires remembering commands and offers limited visibility. Portainer provides a web-based GUI that exposes all Docker functionality visually while still allowing CLI access when needed. Migrating from Docker CLI to Portainer is non-disruptive—your existing containers keep running.

## Installing Portainer Alongside Docker

```bash
# Create a volume for Portainer data
docker volume create portainer_data

# Deploy Portainer (your existing containers are unaffected)
docker run -d \
  --name portainer \
  --restart=always \
  -p 8000:8000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Access Portainer at https://localhost:9443
```

## CLI to Portainer Command Mapping

| Docker CLI Command | Portainer Equivalent |
|--------------------|----------------------|
| `docker ps` | Home > Environments > Containers |
| `docker ps -a` | Containers > Show stopped |
| `docker run ...` | Containers > Add Container |
| `docker stop <name>` | Containers > Select > Stop |
| `docker logs <name>` | Containers > Select > Logs |
| `docker exec -it <name> sh` | Containers > Select > Console |
| `docker inspect <name>` | Containers > Select > Inspect |
| `docker images` | Images (left menu) |
| `docker pull <image>` | Images > Pull Image |
| `docker volume ls` | Volumes (left menu) |
| `docker network ls` | Networks (left menu) |
| `docker-compose up -d` | Stacks > Add Stack |

## Converting docker run to Portainer Container Settings

### Before (CLI)
```bash
docker run -d \
  --name web-app \
  --restart=unless-stopped \
  -p 8080:80 \
  -e NODE_ENV=production \
  -e DATABASE_URL=postgres://db:5432/mydb \
  -v /data/web:/usr/share/nginx/html \
  --network app-network \
  --memory="512m" \
  --cpus="0.5" \
  nginx:1.25
```

### After (Portainer UI steps)
1. Go to **Containers > Add Container**
2. Name: `web-app`
3. Image: `nginx:1.25`
4. Port mapping: `8080:80`
5. Volumes: `/data/web:/usr/share/nginx/html`
6. Environment: `NODE_ENV=production`, `DATABASE_URL=...`
7. Network: `app-network`
8. Resources: Memory limit `512MB`, CPU limit `0.5`
9. Restart policy: `Unless stopped`

## Converting docker-compose to Portainer Stacks

Your existing `docker-compose.yml` files work directly in Portainer:

```bash
# Before: Running with docker-compose
docker-compose -f /opt/myapp/docker-compose.yml up -d

# After: In Portainer
# 1. Go to Stacks > Add Stack
# 2. Name: myapp
# 3. Select "Web editor" or "Git Repository"
# 4. Paste your docker-compose.yml content
# 5. Click "Deploy the stack"
```

## Migrating Existing Containers to Stacks

For containers already running via `docker run`, convert them to stacks for better management:

```bash
# Get the current container configuration
docker inspect my-app > my-app-inspect.json

# Convert to docker-compose format (use composerize tool)
docker run --rm -i ghcr.io/composerize/composerize \
  docker run -d --name my-app -p 8080:80 -e ENV=prod nginx:latest

# Save as docker-compose.yml and import into Portainer as a stack
```

## Setting Up Users and Access Control

Unlike the CLI (which is all-or-nothing), Portainer provides granular access:

```bash
# In Portainer UI:
# Settings > Users > Add User
# - Create accounts for each team member
# - Assign roles: Administrator, Standard User, Read-Only

# Settings > Teams
# - Group users into teams
# - Assign environments per team
```

## Keeping CLI Access

You don't have to give up the CLI entirely:

```bash
# CLI still works alongside Portainer
docker ps                    # Works as before
docker logs my-container     # Works as before

# Use Portainer API for scripting
curl -H "X-API-Key: your-key" \
  https://portainer.example.com/api/endpoints/1/docker/containers/json
```

## Conclusion

Migrating from Docker CLI to Portainer is a zero-downtime transition that adds visibility, collaboration, and access control to your container management. Your existing containers and configurations remain untouched, while Portainer provides a GUI layer on top. The CLI remains available for power users and automation, making this migration purely additive.
