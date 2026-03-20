# How to Optimize Portainer Performance on ARM Devices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, ARM, Raspberry Pi, Performance, Optimization

Description: Learn how to tune Portainer and Docker for optimal performance on resource-constrained ARM devices like Raspberry Pi, Jetson, and other SBCs.

## ARM Devices Running Portainer

Portainer supports ARM32 (armv7) and ARM64 (aarch64) architectures. Common devices:

| Device | Arch | RAM | Notes |
|--------|------|-----|-------|
| Raspberry Pi 3 | armv7 | 1GB | Limited RAM, use arm32v7 images |
| Raspberry Pi 4 | arm64 | 2-8GB | Good performance with 4GB+ |
| Raspberry Pi 5 | arm64 | 4-8GB | Best Pi for Portainer |
| NVIDIA Jetson Nano | arm64 | 4GB | GPU available |
| Orange Pi 5 | arm64 | 4-32GB | High performance ARM |

## Step 1: Use Minimal Portainer Memory Settings

```bash
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  --memory=256m \           # Limit Portainer to 256MB
  --memory-swap=512m \      # Allow 256MB swap
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 2: Enable Swap on the Host

ARM devices often have limited RAM. Swap helps prevent OOM kills:

```bash
# Create a 2GB swapfile
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Reduce swappiness (prefer RAM, use swap only when necessary)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 3: Use a Fast Storage Device

ARM device performance is heavily limited by SD card speed. Use an SSD:

```bash
# Check current storage speed
dd if=/dev/zero of=/tmp/test bs=1M count=1000 oflag=sync
# SD card: ~30 MB/s; USB3 SSD: ~300 MB/s; NVMe (Pi 5): ~900 MB/s

# Move Docker data directory to SSD
sudo systemctl stop docker

# Edit /etc/docker/daemon.json
{
  "data-root": "/mnt/ssd/docker"
}

sudo systemctl start docker
```

## Step 4: Optimize Docker Daemon for ARM

```json
# /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 3
}
```

## Step 5: Use ARM-Optimized Images

Always use ARM-native images, not emulated x86 images:

```bash
# Check image architecture
docker inspect nginx:alpine | jq '.[].Architecture'
# Should be "arm64" or "arm"

# Check what's available
docker manifest inspect nginx:alpine | jq '.manifests[].platform'
```

Use multi-arch images from official sources — they automatically pull the correct architecture.

## Step 6: Limit Portainer Polling

Reduce Portainer's background activity on low-memory devices:

- **Settings → Edge Compute**: Increase heartbeat interval from 15s to 60s
- Avoid running resource-intensive stacks on the same device as Portainer

## Step 7: Reduce Python/Go Overhead in Containers

For Python applications on ARM:

```dockerfile
# Use slim or alpine variants
FROM python:3.12-slim-bookworm

# Pre-compile Python files
RUN python -m compileall /app
```

## Memory Monitoring

```bash
# Watch memory usage
watch -n 5 'free -h && docker stats --no-stream --format "{{.Name}}\t{{.MemUsage}}"'

# Check if swap is being used heavily
vmstat 5 | awk '{print $7, $8}'    # si=swap in, so=swap out
# High values indicate RAM pressure
```

## Conclusion

Portainer runs well on ARM devices with proper tuning. The most impactful changes are adding swap memory, moving Docker storage to a fast SSD, and ensuring you're using native ARM images. On Raspberry Pi 4 with 4GB+ RAM and an SSD, Portainer performs comparably to a low-spec x86 server.
