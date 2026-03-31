# How to Add a Podman Environment to Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Environment, Rootless Containers

Description: Connect Portainer to a Podman environment for managing rootless containers and pods via the Portainer web interface.

## Introduction

Podman is a daemonless container engine that runs containers as rootless processes. Portainer supports Podman environments, allowing you to manage Podman containers, pods, and images through the familiar Portainer UI. This guide covers connecting Portainer to both system-level and user-level Podman installations.

## Prerequisites

- Podman installed on the target host
- Podman socket enabled (system or user)
- Network access between Portainer and the Podman host

## Step 1: Enable Podman Socket

### System-Level Podman Socket (Root)

```bash
# Enable and start the Podman socket

sudo systemctl enable --now podman.socket

# Verify socket is available
ls -la /run/podman/podman.sock

# Test the socket
curl --unix-socket /run/podman/podman.sock http://localhost/v4.0/libpod/info
```

### User-Level Podman Socket (Rootless)

```bash
# Enable user socket
systemctl --user enable --now podman.socket

# Verify socket path
ls -la $XDG_RUNTIME_DIR/podman/podman.sock
# Typically: /run/user/1000/podman/podman.sock

# Or check
podman info | grep -i socket
```

## Step 2: Run Portainer with Podman Socket Access

For Podman running on the same host as Portainer:

```bash
# System Podman socket
podman run -d \
  --name portainer \
  --restart always \
  -p 443:9443 \
  -v /run/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# User Podman socket (rootless)
podman run -d \
  --name portainer \
  --restart always \
  -p 443:9443 \
  -v /run/user/1000/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Note: Portainer expects the socket at `/var/run/docker.sock` internally.

## Step 3: Add Podman Environment via Portainer UI

1. Go to **Environments** → **Add environment**
2. Select **Docker**
3. Choose **Socket** as the connection type
4. The path should be `/var/run/docker.sock` (as mounted)
5. Name the environment (e.g., "Production Podman Host")
6. Click **Connect**

## Using Podman API Over TCP

For remote Podman hosts, expose the Podman API over TCP:

```bash
# On the Podman host - expose API on port 8080
podman system service tcp:0.0.0.0:8080 --time 0 &

# Or with systemd socket activation
cat > /etc/systemd/system/podman-api.socket << 'EOF'
[Unit]
Description=Podman API Socket
Documentation=man:podman-system-service(1)

[Socket]
ListenStream=8080
BindIPv6Only=both

[Install]
WantedBy=sockets.target
EOF

systemctl enable --now podman-api.socket
```

Add in Portainer as Docker environment with URL: `tcp://podman-host:8080`

## Podman Compatibility Notes

Portainer communicates with Podman via the Docker-compatible REST API. Most operations work, but there are some differences:

| Feature | Status |
|---------|--------|
| Container management | Full support |
| Images | Full support |
| Volumes | Full support |
| Networks | Full support |
| Pods | Visible as groups |
| Docker Swarm | Not available (Podman doesn't support Swarm) |
| Docker Compose | Works via Podman Compose compatibility |

## Podman-Compose Compatibility

Portainer can deploy compose stacks via Podman:

```yaml
# example-stack.yml
version: "3.8"

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

Deploy through Portainer's **Stacks** → **Add stack** using the compose editor.

## Checking Connection

```bash
# Test that Portainer can communicate with Podman
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# List containers on Podman environment (endpoint ID 3)
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/3/docker/containers/json?all=true" \
  | python3 -c "import sys,json; [print(c.get('Names')) for c in json.load(sys.stdin)]"
```

## Conclusion

Portainer provides a familiar management interface for Podman deployments, leveraging Podman's Docker-compatible API. The socket-based connection works for local installations, while TCP exposure enables remote Podman management. Note that Swarm-specific features aren't available with Podman - for orchestration, combine Podman with Kubernetes instead.
