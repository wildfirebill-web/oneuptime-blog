# How to Troubleshoot Stack Deployment Failures in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Stacks, Troubleshooting, DevOps

Description: A systematic guide to diagnosing and fixing common stack deployment failures in Portainer.

## Introduction

Stack deployment failures in Portainer can have many causes - from invalid Compose syntax to network conflicts or missing secrets. This guide provides a systematic troubleshooting approach to help you quickly identify and resolve deployment issues.

## Step 1: Read the Deployment Error Message

When a stack deployment fails, Portainer displays an error message in the deployment output panel. Always read this message carefully before investigating further.

Common error patterns and their meanings:

```text
# Image not found

ERROR: pull access denied for myimage, repository does not exist

# Port already in use
Error response from daemon: driver failed programming external connectivity:
Bind for 0.0.0.0:80 failed: port is already allocated

# Invalid Compose syntax
ERROR: The Compose file is invalid because:
  services.api.depends_on contains an invalid type, it should be an object or a list

# Volume driver not found
Error response from daemon: create myvolume: VolumeDriver.Create: no such volume driver
```

## Step 2: Validate Compose File Syntax

Before deploying, validate your Compose file locally:

```bash
# Validate syntax without deploying
docker compose config

# Check a specific file
docker compose -f docker-compose.yml config

# Validate and output the resolved configuration
docker compose config --quiet && echo "Syntax OK"
```

For YAML syntax errors, use a YAML linter:

```bash
# Install yamllint
pip install yamllint

# Check your file
yamllint docker-compose.yml
```

## Step 3: Check Container Exit Codes

If containers start but immediately exit, check their exit codes:

```bash
# List all containers including stopped ones
docker ps -a --filter "label=com.docker.compose.project=mystackname"

# Common exit codes:
# 0  - Clean exit (intentional)
# 1  - Application error
# 137 - Killed by OOM killer (out of memory)
# 139 - Segfault
# 143 - SIGTERM (graceful shutdown)
```

In Portainer: Go to **Containers**, toggle **Show stopped containers**, and check the exit code column.

## Step 4: Examine Container Logs

```bash
# View logs for a crashed container
docker logs <container-id>

# Follow logs to catch startup errors
docker logs -f <container-id>

# Get logs with timestamps
docker logs --timestamps <container-id> 2>&1 | tail -50
```

## Step 5: Common Failure Scenarios and Fixes

### Image Pull Failures

```yaml
# Problem: Private image with no credentials
services:
  app:
    image: private-registry.example.com/myapp:latest  # Requires auth

# Fix: Add registry credentials in Portainer
# Go to: Registries > Add registry > Add your private registry
# Portainer will automatically use credentials when pulling
```

### Environment Variable Not Set

```yaml
services:
  api:
    image: myapi:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}  # Will fail if not set

# Fix in Portainer: Add DATABASE_URL to stack environment variables
# Or use a default value:
      - DATABASE_URL=${DATABASE_URL:-postgresql://localhost/mydb}
```

### Port Conflicts

```bash
# Find what's using port 80
sudo lsof -i :80
# or
sudo netstat -tlnp | grep :80

# Kill the conflicting process or change the port mapping:
```

```yaml
services:
  web:
    ports:
      - "8080:80"  # Changed from 80:80 to avoid conflict
```

### Volume Mount Errors

```bash
# Check if the host path exists
ls -la /path/to/mount

# Create missing directories
mkdir -p /path/to/mount

# Fix permissions
chown 1000:1000 /path/to/mount
```

### Network Name Conflicts

```bash
# List all Docker networks
docker network ls

# Remove a conflicting network (careful: ensure nothing is using it)
docker network rm myapp_default

# Or rename your network in Compose:
```

```yaml
networks:
  myapp-net:        # Use a unique network name
    driver: bridge
```

### Secrets Not Found

```bash
# List available Docker secrets
docker secret ls

# Create a missing secret
echo "mysecretvalue" | docker secret create db_password -
```

## Step 6: Check Resource Constraints

```bash
# Check available disk space
df -h

# Check available memory
free -h

# Check Docker system resource usage
docker system df
```

If disk is full, prune unused resources:

```bash
# Remove unused containers, networks, images
docker system prune

# Also remove unused volumes (careful!)
docker system prune --volumes
```

## Step 7: Debug with an Override File

Create a `docker-compose.override.yml` to add debugging tools:

```yaml
# docker-compose.override.yml (only for debugging)
services:
  api:
    # Override entrypoint to keep container running for inspection
    entrypoint: ["tail", "-f", "/dev/null"]
    # Or add debug environment variables
    environment:
      - LOG_LEVEL=debug
      - DEBUG=true
```

## Step 8: Check Portainer Agent Connectivity

For remote environments, verify the Portainer agent is reachable:

```bash
# Test agent connectivity
curl -k https://agent-host:9001/ping

# Check agent logs
docker logs portainer_agent
```

## Conclusion

Troubleshooting Portainer stack failures requires a systematic approach: read error messages carefully, validate your Compose syntax, examine container logs, and work through common failure scenarios one by one. Most failures fall into a handful of categories - image access, configuration errors, port conflicts, and resource constraints. With this guide, you can quickly pinpoint the root cause and get your stack running.
