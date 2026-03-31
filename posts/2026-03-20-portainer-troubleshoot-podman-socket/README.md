# How to Troubleshoot Podman Socket Connection Issues in Portainer - Socket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Podman, Troubleshooting, Socket, Connectivity, API

Description: Learn how to diagnose and fix Podman socket connection issues when Portainer cannot communicate with the Podman API, including socket permissions and service configuration.

---

Connecting Portainer to a Podman socket involves more configuration than with Docker. This guide covers the common failure modes and their fixes.

## Step 1: Verify the Podman Socket is Active

```bash
# For system-level (rootful) Podman

sudo systemctl status podman.socket
# Should show: Active: active (listening)

# For user-level (rootless) Podman
systemctl --user status podman.socket
# Should show: Active: active (listening)

# List the socket file
ls -la /run/podman/podman.sock          # rootful
ls -la /run/user/$(id -u)/podman/podman.sock  # rootless
```

## Step 2: Test the API Directly

```bash
# Test rootful Podman socket
curl --unix-socket /run/podman/podman.sock \
  http://localhost/v1.41/info

# Expected: JSON response with Podman version info
# If error: socket doesn't exist or service not running
```

## Step 3: Fix "Permission Denied" on the Socket

Portainer's container user may not have access to the Podman socket:

```bash
# Check socket permissions
ls -la /run/podman/podman.sock
# Typical: srw-rw---- 1 root podman

# Option 1: Make socket world-readable (testing only)
sudo chmod 666 /run/podman/podman.sock

# Option 2: Add the group that has socket access to Portainer
# Find the GID of the podman group
getent group podman

# Run Portainer with that supplemental group
docker run -d \
  --name portainer \
  --group-add $(getent group podman | cut -d: -f3) \
  -v /run/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 4: Fix "No Such File or Directory" - Enable the Socket Service

```bash
# Rootful: enable and start the socket
sudo systemctl enable podman.socket
sudo systemctl start podman.socket

# Rootless: enable in user session
systemctl --user enable podman.socket
systemctl --user start podman.socket

# For rootless socket to persist after logout
sudo loginctl enable-linger $(whoami)
```

## Step 5: Check API Version Compatibility

Podman's Docker-compatible API may not implement every endpoint:

```bash
# Check Podman's supported API version
curl --unix-socket /run/podman/podman.sock http://localhost/version | jq .ApiVersion

# Check what Portainer is requesting
docker logs portainer 2>&1 | grep -i "api version\|podman\|v1\." | tail -20
```

If there's a version mismatch, set the API version explicitly:

```bash
# Portainer can be configured to use a specific API version
# Add to Portainer startup: --env DOCKER_API_VERSION=1.41
```

## Step 6: Rootless Socket Inside Docker Container

For rootless Podman with Portainer running in Docker:

```bash
# Map the user-specific socket into the Portainer container
docker run -d \
  --name portainer \
  -v /run/user/$(id -u)/podman/podman.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```
