# How to Connect Portainer to a Podman Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Docker, Self-Hosted, Container Management

Description: Connect Portainer to a Podman socket to manage Podman containers from the Portainer UI, using Podman's Docker-compatible REST API.

## Introduction

Podman provides a Docker-compatible REST API that allows Portainer to manage Podman containers as if they were Docker containers. This guide covers how to enable the Podman socket and connect it to Portainer for a familiar management experience without Docker.

## Prerequisites

- Podman 4.0+ installed
- Portainer CE or BE installed
- Linux system (rootful or rootless Podman supported)

## Step 1: Enable the Podman Socket (Rootful)

```bash
# Enable and start the Podman socket service
sudo systemctl enable --now podman.socket

# Verify the socket is active
sudo systemctl status podman.socket

# Check the socket path
ls -la /run/podman/podman.sock
# Should show: srw-rw---- root root /run/podman/podman.sock
```

## Step 2: Enable the Podman Socket (Rootless)

```bash
# For rootless Podman (user-level socket)
systemctl --user enable --now podman.socket

# Verify
systemctl --user status podman.socket

# Check the socket path
ls -la /run/user/$(id -u)/podman/podman.sock

# Enable lingering so the socket starts without login
sudo loginctl enable-linger $(whoami)
```

## Step 3: Test the Podman Socket

```bash
# Test with rootful socket
curl --unix-socket /run/podman/podman.sock http://d/v5.0/libpod/info | jq .

# Test with Docker-compatible API
curl --unix-socket /run/podman/podman.sock http://d/v1.44/version | jq .

# List containers via Docker-compatible API
curl --unix-socket /run/podman/podman.sock \
  http://d/v1.44/containers/json | jq '.[].Names'
```

## Step 4: Connect Portainer to Podman Socket (Direct)

If Portainer is running on the SAME host as Podman:

```bash
# Mount the Podman socket instead of Docker socket
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /run/podman/podman.sock:/var/run/docker.sock \  # Map Podman socket to Docker socket path
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# For rootless Podman:
docker run -d \
  -p 9000:9000 \
  --name portainer \
  -v /run/user/1000/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Connect Portainer to Remote Podman

For managing Podman on a remote host:

```bash
# Enable the Podman TCP socket on the remote host
# Edit /etc/sysconfig/podman-docker or create systemd override

# Enable via tcp socket
podman system service --time=0 tcp:0.0.0.0:2375 &

# Or create a systemd service
cat > /etc/systemd/system/podman-tcp.socket << 'EOF'
[Unit]
Description=Podman API Socket (TCP)
After=network.target

[Socket]
ListenStream=2375
BindIPv6Only=default

[Install]
WantedBy=sockets.target
EOF

systemctl enable --now podman-tcp.socket
```

In Portainer:
1. Go to **Environments** → **Add Environment**
2. Select **Docker Standalone**
3. Enter URL: `tcp://podman-host:2375`

## Step 6: Use SSH Tunneling for Secure Remote Podman

```bash
# Create SSH tunnel for Podman socket
# On Portainer host:
ssh -L 2375:/run/podman/podman.sock user@podman-host -N &

# Test the tunnel
curl http://localhost:2375/v1.44/version | jq .

# In Portainer, add environment:
# URL: tcp://localhost:2375
```

## Step 7: Configure Portainer Agent for Podman

You can run Portainer Agent against Podman:

```bash
# On the Podman host, run the agent with Podman socket
podman run -d \
  --name portainer-agent \
  --restart=always \
  -p 9001:9001 \
  -v /run/podman/podman.sock:/var/run/docker.sock \
  -v /var/lib/containers/storage:/var/lib/docker \
  docker.io/portainer/agent:latest

# In Portainer server, add Docker Agent environment:
# URL: podman-host:9001
```

## Step 8: Handle Podman-Specific API Differences

Podman's Docker-compatible API has some differences:

```bash
# Check Podman version
curl --unix-socket /run/podman/podman.sock \
  http://d/v1.44/version | jq '.Components[0].Version'

# If you see errors related to Podman-specific endpoints:
# These are expected — Portainer uses Docker-compatible endpoints only

# Check what works and what doesn't
curl --unix-socket /run/podman/podman.sock \
  http://d/v1.44/containers/json  # Should work
curl --unix-socket /run/podman/podman.sock \
  http://d/v1.44/images/json     # Should work
```

## Step 9: Verify Portainer Shows Podman Containers

After connecting, verify Portainer correctly shows Podman containers:

```bash
# Create a Podman container to test
podman run -d --name test-nginx -p 8080:80 nginx:alpine

# In Portainer UI:
# Go to Containers — should show "test-nginx"
# Go to Images — should show nginx:alpine

# Test from Portainer UI:
# - View container logs
# - Open container console (may have limitations)
# - View stats (requires cgroup configuration)
```

## Conclusion

Connecting Portainer to Podman is done by mapping the Podman socket to the path Portainer expects for the Docker socket. Portainer uses the Docker-compatible REST API that Podman provides, which means most features work transparently. The key differences to be aware of are rootful vs rootless Podman socket paths and some API endpoints that are Docker-specific and won't work with Podman.
