# How to Add a Docker Standalone Environment to Portainer via Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Environments, Docker Socket, Configuration

Description: Add a Docker standalone environment to Portainer by mounting the Docker socket, enabling management of the local Docker host.

## Introduction

The simplest way to add a Docker environment to Portainer is via the Docker socket (`/var/run/docker.sock`). When Portainer runs on the same host as Docker, it can communicate with Docker directly via the socket without any additional components. This guide covers setting up Portainer with socket access.

## How Socket Access Works

The Docker socket is a Unix socket that the Docker daemon listens on. When you mount it into the Portainer container, Portainer can send commands to Docker as if it were running natively on the host.

## Step 1: Run Portainer with Socket Access

```bash
docker run -d \
  --name portainer \
  --restart always \
  -p 443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

The key is `-v /var/run/docker.sock:/var/run/docker.sock`.

## Step 2: Docker Compose with Socket

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Step 3: Portainer Automatically Detects the Local Environment

When Portainer first starts with socket access:

1. It detects the Docker socket on startup
2. A "local" environment is automatically created
3. No manual configuration is needed for the local environment

## Step 4: Adding Additional Environments via Socket

If you have a Portainer instance already running and want to add a local environment:

1. Log in to Portainer
2. Go to **Environments** → **Add environment**
3. Select **Docker Standalone**
4. Select **Socket** as the connection type
5. The socket path defaults to `/var/run/docker.sock`
6. Give the environment a name
7. Click **Connect**

## Socket Permissions

The Docker socket requires appropriate permissions:

```bash
# Check socket permissions
ls -la /var/run/docker.sock
# Typically: srw-rw---- 1 root docker 0 Mar 20 10:00 /var/run/docker.sock

# The container user needs read access to the socket
# Option 1: Add portainer to the docker group
sudo usermod -aG docker portainer

# Option 2: Change socket permissions (less secure)
chmod 666 /var/run/docker.sock

# Option 3: Run Portainer as root (default - least recommended for production)
```

For rootless Docker:
```bash
# Rootless Docker socket is typically at:
/run/user/1000/docker.sock

# Mount this socket
docker run -d \
  -v /run/user/1000/docker.sock:/var/run/docker.sock \
  portainer/portainer-ce:latest
```

## Security Considerations

Granting access to the Docker socket is equivalent to root access on the host. Consider these mitigations:

```yaml
# Option: Use Docker socket proxy (limited socket access)
services:
  dockerproxy:
    image: tecnativa/docker-socket-proxy
    environment:
      CONTAINERS: 1      # Allow container list
      IMAGES: 1          # Allow image operations
      NETWORKS: 1        # Allow network operations
      SERVICES: 1        # Allow service operations
      VOLUMES: 1         # Allow volume operations
      POST: 1            # Allow POST (create/delete)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - proxy_internal

  portainer:
    image: portainer/portainer-ce:latest
    environment:
      - DOCKER_HOST=tcp://dockerproxy:2375
    networks:
      - proxy_internal
```

## Verifying the Connection

```bash
# After adding, check the environment status
# In Portainer UI: Environments should show "Up"

# Via API
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

curl -s \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
for env in json.load(sys.stdin):
    print(f'ID={env[\"Id\"]} Name={env[\"Name\"]} Status={env[\"Status\"]}')
"
```

## Conclusion

Socket-based Docker environment access is the simplest and most direct way to connect Portainer to a local Docker host. The docker.sock mount provides full Docker API access without network configuration. For production use, consider using the Docker socket proxy to limit the API surface area exposed to Portainer, reducing the security impact if Portainer were compromised.
