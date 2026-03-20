# How to Fix 'Error 500 on Container Recreation' in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Error 500, Docker, Container Recreation, Debugging

Description: Learn how to diagnose and fix HTTP 500 errors that occur when recreating or duplicating containers in Portainer, including volume conflicts and network issues.

---

A 500 Internal Server Error during container recreation in Portainer is a server-side failure. The error originates from Docker's API - Portainer is surfacing a failure from the Docker daemon. The real cause is almost always in the Docker daemon response.

## Step 1: Check Portainer Logs for the Actual Error

```bash
# Enable debug logging and look for the Docker API error

docker logs portainer 2>&1 | grep -A 5 "error\|500" | tail -50

# For more detail, restart Portainer with debug logging
docker run ... portainer/portainer-ce:latest --log-level DEBUG
```

## Step 2: Identify the Docker API Error

The Portainer UI shows a generic 500 but the underlying Docker error is more specific. Check the browser Network tab (F12) and look at the failing API response body.

Common underlying errors:

| Docker Error | Cause |
|---|---|
| `container name already in use` | Old container not fully removed |
| `network not found` | Referenced network was deleted |
| `volume not found` | Named volume referenced in config no longer exists |
| `port already allocated` | Another container took the port |
| `bind source path does not exist` | Host bind mount path missing |

## Step 3: Fix "Container Name Already in Use"

```bash
# List all containers including stopped ones
docker ps -a | grep <container-name>

# Remove the stopped container with the conflicting name
docker rm <old-container-name>

# Then retry creation in Portainer
```

## Step 4: Fix "Network Not Found"

```bash
# List available networks
docker network ls

# Recreate the missing network
docker network create <missing-network-name>

# Or update the container config in Portainer to use an existing network
```

## Step 5: Fix Missing Bind Mount Path

```bash
# Create the missing host directory
sudo mkdir -p /path/to/missing/directory
sudo chown 1000:1000 /path/to/missing/directory

# Then retry container creation in Portainer
```

## Step 6: Clear the Container Recreation Cache

Portainer caches the container configuration for duplication. If the cached config has invalid entries, clear it by refreshing the environment snapshot:

```bash
docker restart portainer
```

Then try recreating the container from the fresh snapshot.
