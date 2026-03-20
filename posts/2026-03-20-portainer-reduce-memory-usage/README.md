# How to Reduce Portainer Memory Usage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Memory, Performance, Optimization, Docker, Resource Management

Description: Learn how to reduce Portainer's memory footprint by tuning snapshot intervals, garbage collection, log levels, and database compaction.

---

Portainer can consume significant memory when managing many environments with frequent snapshots. This guide covers practical steps to reduce memory usage without sacrificing functionality.

## Diagnosing Memory Usage

Check current Portainer memory consumption:

```bash
# Real-time memory stats

docker stats portainer --no-stream --format "{{.MemUsage}}"

# Detailed memory breakdown
docker exec -it portainer cat /proc/1/status | grep -E "VmRSS|VmSwap|VmPeak"
```

The main memory consumers in Portainer are:

1. **DockerSnapshotRaw** - raw Docker API responses cached in BoltDB
2. **Go runtime** - garbage collector heap
3. **Active HTTP connections** - WebSocket sessions for logs/console
4. **BoltDB mmap** - memory-mapped database file

## Step 1: Increase Snapshot Interval

Each snapshot stores the full state of an environment in memory. Reducing snapshot frequency directly reduces memory pressure:

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 300    # 5 minutes instead of 60 seconds
```

## Step 2: Tune Go Garbage Collector

The Go runtime's garbage collector can be tuned to trade CPU for lower memory:

```bash
docker run -d \
  --name portainer \
  -e GOGC=50 \           # Run GC more aggressively (default 100)
  -e GOMEMLIMIT=400MiB \ # Hard memory limit (Go 1.19+)
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

`GOGC=50` triggers garbage collection when heap grows to 50% above the last collection (default is 100%), reducing peak memory at the cost of slightly more CPU.

## Step 3: Compact the BoltDB Database

A large database file increases the memory footprint due to mmap:

```bash
# Check current database size
docker exec -it portainer du -sh /data/portainer.db

# Stop and compact
docker stop portainer
docker run --rm -v portainer_data:/data \
  portainer/portainer-ce:latest --compact-db
docker start portainer

# Compare size after compaction
docker exec -it portainer du -sh /data/portainer.db
```

## Step 4: Reduce Log Level

Lower log levels reduce string allocation overhead:

```bash
docker run -d \
  --name portainer \
  portainer/portainer-ce:latest \
  --log-level warn   # Only log warnings and errors
```

## Step 5: Set Memory Limit to Prevent OOM Kill

Set a memory limit to prevent Portainer from consuming all available RAM:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    mem_limit: 512m
    memswap_limit: 512m   # Disable swap extension
```

If Portainer is OOM-killed at 512 MB, increase to 768 MB or 1 GB for large deployments.

## Step 6: Remove Unused Environments

Each connected environment (endpoint) consumes memory for its snapshot. Disconnect unused or decommissioned environments:

1. In Portainer, go to **Environments**.
2. Click the trash icon next to unused environments.
3. Or disable auto-snapshot for low-priority environments.

## Memory Usage by Scale

| Environments | Containers | Expected Memory |
|--------------|------------|-----------------|
| 1–5          | <50        | 64–128 MB       |
| 5–15         | 50–200     | 128–256 MB      |
| 15–30        | 200–500    | 256–512 MB      |
| 30+          | 500+       | 512 MB – 1 GB   |

## Monitoring Memory Trends

Track memory over time to catch gradual leaks:

```bash
# Log memory every 5 minutes
while true; do
  echo "$(date +%H:%M) $(docker stats portainer --no-stream --format "{{.MemUsage}}")"
  sleep 300
done >> /var/log/portainer-memory.log
```
