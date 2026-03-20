# How to Manage Containers via the Portainer API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Containers, Automation, Docker

Description: Learn how to list, start, stop, restart, and inspect Docker containers using the Portainer REST API.

## Container API Endpoints

Portainer proxies Docker API calls through its endpoints. Container operations go through:
```
/api/endpoints/{endpointId}/docker/containers/{action}
```

## Listing Containers

```bash
PORTAINER_URL="https://portainer.mycompany.com"
API_TOKEN="your_access_token"
ENDPOINT_ID=1

# List all running containers
curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id[0:12], name: .Names[0], image: .Image, status: .Status}]'

# List all containers including stopped ones
curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?all=true" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '[.[] | {id: .Id[0:12], name: .Names[0], status: .Status}]'
```

## Inspecting a Container

```bash
# Get detailed container information
CONTAINER_ID="abc123def456"

curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/json" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '{
    name: .Name,
    image: .Config.Image,
    state: .State.Status,
    ip: .NetworkSettings.IPAddress,
    mounts: [.Mounts[].Destination]
  }'
```

## Starting and Stopping Containers

```bash
# Start a stopped container
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/start" \
  -H "Authorization: Bearer ${API_TOKEN}"
# Returns 204 No Content on success

# Stop a running container
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/stop?t=30" \
  -H "Authorization: Bearer ${API_TOKEN}"
# t=30: wait 30 seconds before killing

# Restart a container
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/restart" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Getting Container Logs

```bash
# Get last 100 lines of logs
curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/logs?stdout=true&stderr=true&tail=100" \
  -H "Authorization: Bearer ${API_TOKEN}"
```

## Container Stats

```bash
# Get resource usage stats (one-shot, not streaming)
curl -s "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${CONTAINER_ID}/stats?stream=false" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq '{
    cpu_percent: (100 * (.cpu_stats.cpu_usage.total_usage - .precpu_stats.cpu_usage.total_usage) / (.cpu_stats.system_cpu_usage - .precpu_stats.system_cpu_usage) * (.cpu_stats.online_cpus // 1)),
    memory_usage_mb: (.memory_stats.usage / 1048576),
    memory_limit_mb: (.memory_stats.limit / 1048576)
  }'
```

## Bulk Operations Script

```bash
#!/bin/bash
# Restart all containers matching a name pattern

NAME_FILTER="my-app"

# Get container IDs matching the filter
CONTAINER_IDS=$(curl -s \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?all=true" \
  -H "Authorization: Bearer ${API_TOKEN}" | \
  jq -r --arg filter "$NAME_FILTER" \
  '.[] | select(.Names[0] | contains($filter)) | .Id')

for ID in $CONTAINER_IDS; do
  echo "Restarting container: ${ID:0:12}"
  curl -s -X POST \
    "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/${ID}/restart" \
    -H "Authorization: Bearer ${API_TOKEN}"
done
```

## Creating a Container

```bash
# Create and start a new container
curl -X POST \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/create?name=my-nginx" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "nginx:alpine",
    "ExposedPorts": {"80/tcp": {}},
    "HostConfig": {
      "PortBindings": {"80/tcp": [{"HostPort": "8080"}]},
      "RestartPolicy": {"Name": "unless-stopped"}
    }
  }'
```

## Conclusion

The Portainer container management API mirrors the Docker API, making it easy to build automation around container operations. It adds authentication and audit logging on top of the raw Docker API, making it safer for team environments.
