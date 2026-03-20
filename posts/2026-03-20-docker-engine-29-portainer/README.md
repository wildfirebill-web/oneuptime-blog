# How to Fix Docker Engine 29 Compatibility Issues with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Docker Engine 29, Compatibility

Description: Resolve compatibility issues between Portainer and Docker Engine 29, including API version mismatches, deprecated features, and configuration changes.

## Introduction

Docker Engine 29 introduced changes to the API, networking defaults, and deprecated several older behaviors. If you upgraded to Docker Engine 29.x and are experiencing Portainer issues — errors in the UI, missing containers, network creation failures — this guide covers the known compatibility issues and their fixes.

## Known Issues with Docker Engine 29 and Portainer

1. API version negotiation changes
2. Default bridge network behavior changes
3. `--network-alias` deprecation warnings appearing in Portainer
4. IPv6 now enabled by default, causing unexpected network behavior
5. BuildKit changes affecting image builds from Portainer

## Step 1: Check Your Docker Engine Version

```bash
# Check Docker server version and API version
docker version

# Example output:
# Server: Docker Engine - Community
#  Engine:
#   Version:          29.0.1
#   API version:      1.45 (minimum version 1.24)
```

## Step 2: Verify Portainer Version Compatibility

Portainer releases track Docker API versions. Check the compatibility matrix:

```bash
# Check running Portainer version
docker exec portainer /app/portainer --version

# Or via the UI: Help → About

# Portainer 2.21+ supports Docker Engine 29 API
# If you're on an older version, update Portainer
```

## Step 3: Update Portainer to Latest

```bash
# Pull the latest Portainer image
docker pull portainer/portainer-ce:latest

# Stop and remove old container
docker stop portainer
docker rm portainer

# Restart with latest image (data volume preserved)
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 4: Fix IPv6 Issues

Docker Engine 29 enables IPv6 by default. This can cause network creation to fail in Portainer if IPv6 is not configured on the host:

```bash
# Check if IPv6 is causing issues
docker network create test-net 2>&1

# Disable IPv6 in Docker daemon if not needed
cat > /etc/docker/daemon.json << 'EOF'
{
  "ipv6": false
}
EOF

sudo systemctl restart docker
```

If you need IPv6 support:

```bash
# Configure proper IPv6 in Docker daemon
cat > /etc/docker/daemon.json << 'EOF'
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80"
}
EOF

sudo systemctl restart docker
```

## Step 5: Fix BuildKit Issues

Docker Engine 29 uses BuildKit by default for all builds. Some Portainer image build workflows may encounter issues:

```bash
# In the Docker daemon config, you can control BuildKit behavior
cat > /etc/docker/daemon.json << 'EOF'
{
  "features": {
    "buildkit": true
  },
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  }
}
EOF

sudo systemctl restart docker
```

In Portainer, when building images:
1. Go to **Images** → **Build**
2. If you see BuildKit-related errors, check the **Advanced** options
3. Ensure the Dockerfile syntax is compatible with BuildKit

## Step 6: Fix Container Stats Errors

Docker Engine 29 changed the stats API format. If Portainer shows "Unable to retrieve container stats":

```bash
# Test the stats API directly
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.45/containers/portainer/stats?stream=false | jq .

# If this works but Portainer doesn't show stats, update Portainer
```

## Step 7: Fix Network Listing Issues

Docker Engine 29 changed how networks with `--internal` flag are reported:

```bash
# List networks via API directly to compare
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.45/networks | jq '.[].Name'

# If some networks appear in API but not in Portainer
# This is a Portainer version issue — update to 2.21+
```

## Step 8: Fix Image Pull Errors

Docker Engine 29 introduced stricter image registry authentication handling:

```bash
# Test image pull via CLI to isolate the issue
docker pull nginx:latest

# If CLI pull works but Portainer fails
# Check Portainer registry credentials: Settings → Registries
# Re-enter credentials for any configured registries
```

## Step 9: Configure Docker Daemon for Portainer Compatibility

Create an optimized `/etc/docker/daemon.json` for use with Portainer on Docker Engine 29:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "features": {
    "buildkit": true
  },
  "live-restore": true
}
```

```bash
sudo systemctl restart docker
```

## Conclusion

Docker Engine 29 compatibility with Portainer is primarily addressed by updating Portainer to version 2.21 or later, which was built to support the Docker Engine 29 API. IPv6 default enablement and BuildKit changes are the most common secondary causes of unexpected behavior — both are easily addressed with daemon configuration changes.
