# How to Fix Portainer Memory Issues on Low-Resource Hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Performance, Self-Hosted, Low-Resource

Description: Optimize Portainer's memory usage on VPS instances or edge devices with limited RAM using resource limits, snapshot tuning, and agent deployment strategies.

## Introduction

Portainer runs well on most modern servers, but on low-resource hosts (1-2 GB RAM VPS, Raspberry Pi, edge devices), it can consume significant memory — especially when managing many containers or large Docker environments. This guide covers techniques to reduce Portainer's footprint.

## Understanding Portainer's Memory Usage

Portainer uses memory for:
- The web server and API
- Docker environment snapshots (stored in-memory cache)
- WebSocket connections for live stats
- The embedded BoltDB database
- Kubernetes cluster caches (if applicable)

## Step 1: Check Current Memory Usage

```bash
# Check Portainer's current memory consumption
docker stats portainer --no-stream

# Output example:
# NAME       CPU %  MEM USAGE / LIMIT  MEM %
# portainer  0.2%   245MB / 1GB        24.5%

# Monitor over time
docker stats portainer
```

## Step 2: Set Memory Limits on Portainer Container

Prevent Portainer from consuming all available RAM:

```bash
# Recreate with memory limit (adjust based on your available RAM)
docker stop portainer && docker rm portainer

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  --memory="256m" \
  --memory-swap="512m" \
  --memory-reservation="128m" \
  --cpus="0.5" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Or in Docker Compose:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.50'
        reservations:
          memory: 128M
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
```

## Step 3: Increase the Snapshot Interval

By default, Portainer snapshots your Docker environment frequently. Reducing this frequency saves memory:

```bash
# Start Portainer with a longer snapshot interval (in seconds)
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval=300   # Snapshot every 5 minutes instead of default
```

## Step 4: Use Portainer Agent Instead of Direct Docker Socket

On resource-constrained hosts managing remote Docker environments, use the lightweight Agent instead of the full Portainer server:

```bash
# On the low-resource host, run ONLY the agent (very lightweight)
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  --memory="64m" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# Run the full Portainer server on a more powerful host
# and connect to this agent remotely
```

The Portainer Agent typically uses only 20-50 MB RAM, compared to 200+ MB for the full server.

## Step 5: Reduce the Number of Managed Environments

Each environment Portainer actively monitors consumes memory for snapshots:

1. In Portainer UI, go to **Environments**
2. Remove environments you're not actively using
3. Or set them to **inactive** if that's available in your version

## Step 6: Enable Swap on Low-RAM Hosts

If you can't reduce Portainer's memory usage further, ensure swap is available:

```bash
# Check current swap
free -h

# Create a swap file if none exists
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Reduce swappiness for SSD/flash storage (0-10 recommended)
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 7: Use Portainer Agent on Raspberry Pi / ARM

For ARM-based devices like Raspberry Pi:

```bash
# The standard image supports ARM, but ensure you're using the right variant
# For Raspberry Pi (ARM64):
docker pull portainer/portainer-ce:latest  # Multi-arch, includes ARM64

# For older Raspberry Pi (ARM32):
docker pull portainer/portainer-ce:latest  # Check manifest for ARM/v7

# Run with strict memory limits appropriate for Pi
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  --memory="192m" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval=600
```

## Step 8: Monitor with cAdvisor

To track memory usage trends and identify spikes:

```bash
# Deploy cAdvisor for container metrics
docker run -d \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --name=cadvisor \
  --memory="128m" \
  gcr.io/cadvisor/cadvisor:latest
```

## Step 9: Tune the BoltDB Database

On long-running installations, the BoltDB database can grow large:

```bash
# Compact the database using Portainer's built-in flag
docker stop portainer && docker rm portainer

docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Restart normally
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Conclusion

Portainer's memory usage on low-resource hosts is manageable with a combination of container memory limits, increased snapshot intervals, and using the lightweight Agent on resource-constrained nodes. For the most minimal footprint, run just the Portainer Agent on edge devices and connect them to a central Portainer server on a more powerful machine.
