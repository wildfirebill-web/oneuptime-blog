# How to Manage Containers via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Docker, Containers, Automation

Description: Learn how to list, create, start, stop, inspect, and remove Docker containers via the Portainer REST API, using Portainer as a Docker API proxy.

## Introduction

The Portainer API proxies the Docker Engine API, allowing you to manage containers programmatically through Portainer's authentication and access control layer. This is ideal for automation scripts that need container lifecycle management with Portainer's RBAC enforcement.

## Prerequisites

- Portainer CE or BE with a Docker environment
- Valid JWT token or API access token
- Target endpoint (environment) ID
- `curl` and `jq` installed

## Step 1: List Containers

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-token"
ENDPOINT_ID=1

# List running containers
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json" | \
  jq '.[] | {id: .Id[0:12], name: .Names[0], image: .Image, status: .Status}'

# List ALL containers (including stopped)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?all=true" | \
  jq '.[] | {id: .Id[0:12], name: .Names[0], image: .Image, status: .State}'

# Filter containers by label
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?filters=%7B%22label%22%3A%5B%22app%3Dmyapp%22%5D%7D" | jq .
```

## Step 2: Inspect a Container

```bash
CONTAINER_ID="abc123def456"

# Get full container configuration and state
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/json" | jq '{
    id: .Id[0:12],
    name: .Name,
    image: .Config.Image,
    state: .State.Status,
    started: .State.StartedAt,
    ports: .NetworkSettings.Ports,
    mounts: .Mounts,
    env: .Config.Env
  }'
```

## Step 3: Create a Container

```bash
# Create a new container
CREATE_RESPONSE=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/create?name=my-nginx" \
  -d '{
    "Image": "nginx:1.25",
    "ExposedPorts": {
      "80/tcp": {}
    },
    "HostConfig": {
      "PortBindings": {
        "80/tcp": [{"HostPort": "8080"}]
      },
      "RestartPolicy": {
        "Name": "unless-stopped"
      },
      "Binds": ["/host/path:/container/path"]
    },
    "Env": [
      "NGINX_HOST=example.com",
      "NGINX_PORT=80"
    ],
    "Labels": {
      "app": "myapp",
      "env": "production"
    }
  }')

CONTAINER_ID=$(echo $CREATE_RESPONSE | jq -r '.Id')
echo "Container created with ID: $CONTAINER_ID"
```

## Step 4: Start, Stop, and Restart Containers

```bash
CONTAINER_ID="abc123def456"

# Start a container
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/start"

echo "Container started."

# Stop a container (graceful, 10s timeout)
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/stop?t=10"

echo "Container stopped."

# Restart a container
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/restart?t=10"

echo "Container restarted."

# Kill a container (immediate, SIGKILL)
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/kill"
```

## Step 5: Get Container Logs

```bash
CONTAINER_ID="abc123def456"

# Get last 100 log lines
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/logs?stdout=true&stderr=true&tail=100"

# Get logs with timestamps
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/logs?stdout=true&stderr=true&timestamps=true&tail=50"
```

## Step 6: Execute a Command in a Container

```bash
CONTAINER_ID="abc123def456"

# Create an exec instance
EXEC_ID=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/exec" \
  -d '{
    "AttachStdout": true,
    "AttachStderr": true,
    "Cmd": ["sh", "-c", "ls -la /app && df -h"]
  }' | jq -r '.Id')

echo "Exec ID: $EXEC_ID"

# Start the exec
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/exec/${EXEC_ID}/start" \
  -d '{"Detach": false, "Tty": false}'
```

## Step 7: Remove a Container

```bash
CONTAINER_ID="abc123def456"

# Remove stopped container
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}"

# Force remove running container
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}?force=true"

# Remove container and its anonymous volumes
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}?v=true&force=true"
```

## Step 8: Get Container Stats

```bash
CONTAINER_ID="abc123def456"

# Get a one-time stats snapshot (stream=false)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/stats?stream=false" | \
  jq '{
    cpu_percent: (.cpu_stats.cpu_usage.total_usage / .cpu_stats.system_cpu_usage * 100),
    memory_mb: (.memory_stats.usage / 1048576)
  }'
```

## Conclusion

The Portainer API provides full Docker container lifecycle management with the security benefits of Portainer's authentication and RBAC layer. Use it to automate container creation, monitor running workloads, retrieve logs for debugging, and clean up unused containers — all without requiring direct Docker socket access. This is particularly valuable when you want to centralize access control through Portainer rather than distributing Docker socket credentials.
