# How to Configure Portainer SSD Requirements for Best Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, SSD, Storage, Infrastructure

Description: Configure Portainer and Docker storage on SSDs to achieve optimal database performance, fast image pulls, and responsive UI in container management environments.

## Introduction

Portainer's embedded boltdb database performs significantly better on SSDs than spinning disks. Docker's overlay2 storage driver also benefits from SSD I/O for layer operations during builds and container startups. This guide covers identifying storage bottlenecks, moving Portainer and Docker storage to SSD, and benchmarking the improvement.

## Step 1: Identify Current Storage Configuration

```bash
# Check what storage device Portainer data is on

docker inspect portainer --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{end}}'
# Output: /var/lib/docker/volumes/portainer_data/_data -> /data

# Find which disk that path is on
df -h /var/lib/docker/volumes/portainer_data/_data

# Check if it's SSD or spinning disk
lsblk -o NAME,ROTA,SIZE,TYPE,MOUNTPOINT
# ROTA=0: SSD (Solid State Drive)
# ROTA=1: HDD (Hard Disk Drive - spinning)

# Check I/O scheduler (important for performance)
cat /sys/block/sda/queue/scheduler
# SSD optimal: [none] or [mq-deadline]
# HDD optimal: [mq-deadline] or [bfq]

# Benchmark current I/O performance
dd if=/dev/zero of=/var/lib/docker/volumes/portainer_data/_data/test \
  bs=1M count=1000 oflag=direct 2>&1 | tail -1
# Remove the test file after
rm /var/lib/docker/volumes/portainer_data/_data/test
```

## Step 2: Mount SSD for Portainer Data

```bash
# Assuming you have a dedicated SSD at /dev/sdb
# Format and mount the SSD

# Format with ext4
sudo mkfs.ext4 -E lazy_itable_init=0,lazy_journal_init=0 /dev/sdb

# Create mount point
sudo mkdir -p /opt/ssd

# Mount with performance-optimized options
sudo mount -o noatime,nodiratime,data=writeback /dev/sdb /opt/ssd

# Make persistent across reboots
echo "/dev/sdb /opt/ssd ext4 noatime,nodiratime,data=writeback 0 2" | \
  sudo tee -a /etc/fstab

# Create Portainer directory on SSD
sudo mkdir -p /opt/ssd/portainer/data
sudo chown -R 1000:1000 /opt/ssd/portainer
```

## Step 3: Configure Portainer on SSD Storage

```yaml
# docker-compose.yml - Portainer with SSD storage
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    command:
      - "--snapshot-interval=300"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # Bind mount to SSD directory (not named volume)
      - /opt/ssd/portainer/data:/data
    ports:
      - "9443:9443"
    # Prevent writes from being buffered (ensures durability)
    environment:
      - PORTAINER_DATA=/data
```

## Step 4: Move Docker's Storage Root to SSD

```json
// /etc/docker/daemon.json - Move Docker storage to SSD
{
  "data-root": "/opt/ssd/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

```bash
# Move existing Docker data to SSD
sudo systemctl stop docker

# Copy all Docker data to SSD
sudo rsync -aP /var/lib/docker/ /opt/ssd/docker/

# Start Docker with new location
sudo systemctl start docker

# Verify
docker info | grep "Docker Root Dir"
# Should show: Docker Root Dir: /opt/ssd/docker
```

## Step 5: Set Optimal I/O Scheduler for SSD

```bash
# Set I/O scheduler for optimal SSD performance
# Option 1: none (best for NVMe SSDs)
echo "none" | sudo tee /sys/block/nvme0n1/queue/scheduler

# Option 2: mq-deadline (good for SATA SSDs)
echo "mq-deadline" | sudo tee /sys/block/sda/queue/scheduler

# Make persistent across reboots
cat > /etc/udev/rules.d/60-ioscheduler.rules << 'EOF'
# NVMe SSDs - use none scheduler
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
# SATA SSDs (rotational=0) - use mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
# HDDs (rotational=1) - use bfq
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
EOF

# Disable read-ahead for SSDs (not needed)
sudo blockdev --setra 0 /dev/sda
```

## Step 6: Benchmark Before and After

```bash
# Benchmark Portainer database operations
# Before SSD: measure boltdb write performance

# Install fio for proper I/O benchmarking
apt-get install -y fio

# Sequential read test (simulates snapshot reads)
fio --name=seq-read \
  --directory=/opt/ssd/portainer/data \
  --rw=read \
  --bs=4k \
  --size=1G \
  --numjobs=1 \
  --time_based \
  --runtime=30 \
  --output-format=normal

# Random write test (simulates boltdb writes)
fio --name=rand-write \
  --directory=/opt/ssd/portainer/data \
  --rw=randwrite \
  --bs=4k \
  --size=1G \
  --numjobs=4 \
  --iodepth=32 \
  --time_based \
  --runtime=30

# Compare Portainer API response times before and after
for i in $(seq 1 10); do
  time curl -s \
    -H "Authorization: Bearer $TOKEN" \
    "https://portainer.example.com/api/endpoints" > /dev/null
done
```

## Conclusion

SSD storage for Portainer's database is one of the most impactful infrastructure changes you can make. boltdb is write-heavy - every snapshot creates database transactions - and spinning disk latency makes these writes the bottleneck. Moving to SSD typically reduces API response times by 50-80% in environments with many containers. For Docker's overlay2 storage, SSD dramatically speeds up container startups and image builds. The NVMe `none` I/O scheduler eliminates kernel-level scheduling overhead for drives that have their own intelligent queuing. Benchmark before and after to quantify the improvement for your specific workload.
