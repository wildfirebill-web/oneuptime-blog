# How to Connect Portainer to a Podman Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Docker Socket, Linux, Container Management, Rootless

Description: Learn how to connect Portainer to a Podman socket to manage Podman containers through the Portainer UI, using Podman's Docker-compatible API.

---

Podman provides a Docker-compatible REST API that Portainer can use. By pointing Portainer at the Podman socket, you get the same management interface for Podman containers as for Docker.

## Prerequisites

- Podman installed on the host (version 3.x or later)
- Podman socket service enabled
- Portainer running

## Step 1: Enable the Podman Socket

```bash
# For rootful Podman (runs as root)

sudo systemctl enable --now podman.socket
sudo systemctl status podman.socket

# Verify the socket exists
ls -la /run/podman/podman.sock

# For rootless Podman (runs as a normal user)
systemctl --user enable --now podman.socket
ls -la /run/user/$(id -u)/podman/podman.sock
```

## Step 2: Test the Podman API

```bash
# Test that the Podman socket responds to Docker API calls
curl --unix-socket /run/podman/podman.sock http://localhost/version

# Should return JSON with Podman version info
# The API format is Docker-compatible
```

## Step 3: Configure Portainer to Use the Podman Socket

**Option A: Mount the Podman socket in the Portainer container**

```bash
docker run -d \
  --name portainer \
  --restart=always \
  -p 9000:9000 \
  -v /run/podman/podman.sock:/var/run/docker.sock \  # Map Podman socket to Docker socket path
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

**Option B: Add Podman as a separate environment via the API socket**

1. In Portainer go to **Environments > Add Environment**.
2. Choose **Docker Standalone > API URL**.
3. Set the socket path to `unix:///run/podman/podman.sock`.

## Step 4: Fix Podman Socket Permissions

Portainer runs as root; the Podman socket may restrict access:

```bash
# Check socket permissions
ls -la /run/podman/podman.sock

# Fix permissions if needed
sudo chmod 666 /run/podman/podman.sock

# Or add the portainer container user to the podman group
sudo usermod -aG podman $(whoami)
```

## Limitations

Portainer with Podman has some differences from Docker:

- Pods (Podman-specific) are not shown natively in Portainer
- Volume management may differ slightly
- Build functionality depends on Podman's build API compatibility

Despite these, container start/stop/exec/logs and most stack operations work correctly.
