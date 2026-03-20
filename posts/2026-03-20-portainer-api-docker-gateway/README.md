# How to Use the Portainer API as a Docker API Gateway - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API, Docker, Gateway, Security

Description: Learn how to use Portainer as a secure gateway to the Docker Engine API, enabling controlled Docker API access with authentication and RBAC without exposing the Docker socket directly.

## Introduction

Exposing the Docker socket directly is a significant security risk, as anyone with socket access has root-equivalent control over the host. Portainer acts as a secure API gateway in front of Docker, proxying Docker API requests through its own authentication and authorization layer. This guide covers how to use this gateway pattern effectively.

## Prerequisites

- Portainer CE or BE installed
- A Docker environment connected to Portainer
- Valid authentication credentials for Portainer
- Understanding of Docker Engine API

## Architecture

```bash
Client → Portainer API Gateway → Docker Socket → Container Runtime
         (authentication,        (full Docker
          RBAC, audit)           API access)
```

Portainer proxies requests from:
```text
/api/endpoints/{id}/docker/{path}
```
to the underlying Docker Engine at `docker.sock` (or TCP endpoint).

## Step 1: Understanding the Proxy URL Pattern

All Docker API calls through Portainer follow this pattern:

```text
https://portainer.example.com/api/endpoints/{endpointId}/docker/{docker-api-path}
```

For example:
```bash
Docker direct:   GET /containers/json
Via Portainer:   GET /api/endpoints/1/docker/containers/json
```

## Step 2: Common Docker API Operations via Portainer Gateway

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your-auth-token"
ENDPOINT_ID=1

# Docker info (equivalent to: docker info)

curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/info" | jq .

# Docker version (equivalent to: docker version)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/version" | jq .

# List containers (equivalent to: docker ps -a)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/json?all=true" | jq .

# List images (equivalent to: docker images)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/json" | jq .

# List volumes (equivalent to: docker volume ls)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/volumes" | jq .

# List networks (equivalent to: docker network ls)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/networks" | jq .

# System disk usage (equivalent to: docker system df)
curl -s -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/system/df" | jq .
```

## Step 3: Use Portainer as a Docker Remote Context

Configure your local Docker CLI to use Portainer as a remote Docker context:

```bash
# The Portainer Docker API endpoint URL
DOCKER_HOST_URL="https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/docker"

# Create a Docker context that uses Portainer
# Note: Docker CLI contexts use TLS, so use docker CLI directly with env vars

# Set DOCKER_HOST to the Portainer proxy
export DOCKER_HOST="tcp://portainer.example.com:9443"

# For Docker SDK in Python
import docker
import requests

class PortainerDockerClient:
    """Docker client that routes through Portainer API."""

    def __init__(self, portainer_url, api_key, endpoint_id):
        self.base_url = f"{portainer_url}/api/endpoints/{endpoint_id}/docker"
        self.headers = {"X-API-Key": api_key}
        self.session = requests.Session()
        self.session.headers.update(self.headers)

    def list_containers(self, all=False):
        resp = self.session.get(f"{self.base_url}/containers/json",
                               params={"all": str(all).lower()})
        resp.raise_for_status()
        return resp.json()

    def start_container(self, container_id):
        resp = self.session.post(f"{self.base_url}/containers/{container_id}/start")
        return resp.status_code == 204

    def pull_image(self, image_name, tag="latest"):
        resp = self.session.post(f"{self.base_url}/images/create",
                                params={"fromImage": image_name, "tag": tag},
                                stream=True)
        for line in resp.iter_lines():
            if line:
                print(line.decode())
```

## Step 4: Pull Images via Portainer Gateway

```bash
# Pull an image on a remote Docker host via Portainer
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/create?fromImage=nginx&tag=1.25"

# Pull with authentication (for private registries)
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Registry-Auth: $(echo '{"username":"user","password":"pass","serveraddress":"registry.example.com"}' | base64)" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/create?fromImage=registry.example.com/myapp&tag=latest"
```

## Step 5: Create Networks via Gateway

```bash
# Create a custom bridge network
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/networks/create" \
  -d '{
    "Name": "app-network",
    "Driver": "bridge",
    "IPAM": {
      "Config": [{"Subnet": "172.30.0.0/16"}]
    },
    "Labels": {
      "project": "myapp"
    }
  }' | jq .
```

## Step 6: Prune Resources via Gateway

```bash
# Remove stopped containers
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/containers/prune" | jq .

# Remove unused images
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/images/prune" | jq .

# Remove unused volumes
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/volumes/prune" | jq .

# Remove unused networks
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  "${PORTAINER_URL}/api/endpoints/${ENDPOINT_ID}/docker/networks/prune" | jq .
```

## Security Benefits of the Gateway Pattern

1. **No Docker socket exposure**: The socket stays on the server; clients never get direct access
2. **Centralized authentication**: All requests authenticated via Portainer credentials
3. **Audit logging**: Every API call is logged by Portainer (BE)
4. **RBAC enforcement**: Users can only operate within their permitted environments
5. **TLS termination**: All traffic encrypted between client and Portainer

## Conclusion

Using Portainer as a Docker API gateway provides a secure, authenticated alternative to direct Docker socket access. Teams can perform the same Docker operations they're used to while Portainer enforces access controls, logs activity, and eliminates the need to distribute Docker socket access credentials. This pattern is particularly valuable for multi-team environments where different teams should have different levels of Docker access.
