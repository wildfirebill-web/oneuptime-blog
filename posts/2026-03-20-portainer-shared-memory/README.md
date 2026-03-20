# How to Set Shared Memory Size for Containers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Containers, Performance, Linux

Description: Learn how to configure the shared memory (shm_size) for Docker containers in Portainer, which is required by browsers, databases, and AI/ML workloads.

## Introduction

Docker containers have a default `/dev/shm` (shared memory) size of 64 MB. Many applications — web browsers running in headless mode, PostgreSQL, certain machine learning frameworks, and video processing tools — require significantly more shared memory to function correctly. Portainer lets you configure this limit when creating a container.

## Prerequisites

- Portainer installed with a connected Docker environment

## Why Shared Memory Matters

Shared memory (`/dev/shm`) is a tmpfs filesystem used for inter-process communication (IPC) and shared memory segments. When it's too small:

- Chromium/Chrome headless crashes with "No space left on device"
- PostgreSQL may fail to create shared memory segments
- PyTorch DataLoader workers crash when using multiprocessing
- Some video encoding tools fail

## Step 1: Set Shared Memory in Portainer

1. Navigate to **Containers > Add container**.
2. Scroll to the **Runtime & Resources** section.
3. Find the **Shared memory size** field.
4. Enter the size in MB.

```
Shared memory size: 256   (MB)
# or
Shared memory size: 2048  (MB for ML workloads)
```

## Step 2: Common Shared Memory Requirements

### Chromium/Chrome Headless

The most common reason to increase shared memory:

```yaml
# docker-compose.yml
services:
  chrome:
    image: selenium/standalone-chrome:latest
    shm_size: '2g'   # 2 GB shared memory for Chrome
    # Without this, Chrome crashes with "--disable-dev-shm-usage workaround"
```

Or with the workaround flag (avoids the need for increased shm):

```yaml
services:
  puppeteer:
    image: node:20-alpine
    command: node puppeteer-script.js
    environment:
      - PUPPETEER_ARGS=--no-sandbox,--disable-dev-shm-usage
```

The `--disable-dev-shm-usage` flag makes Chrome write to `/tmp` instead, avoiding the shm limit but potentially being slower.

### PostgreSQL

```yaml
services:
  postgres:
    image: postgres:15-alpine
    shm_size: '256m'   # Increase from 64 MB default
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_SHARED_BUFFERS=128MB  # Should be <= shm_size
```

PostgreSQL uses shared memory for its shared buffer pool. The `shm_size` must be at least as large as `shared_buffers`.

### PyTorch / ML Workloads

```yaml
services:
  ml-training:
    image: pytorch/pytorch:2.2.0-cuda12.1-cudnn8-runtime
    shm_size: '8g'   # 8 GB for large DataLoader worker pools
    environment:
      - CUDA_VISIBLE_DEVICES=0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

PyTorch DataLoader with `num_workers > 0` uses shared memory to pass data between workers and the main process.

### Apache Spark

```yaml
services:
  spark-worker:
    image: apache/spark:3.5.0
    shm_size: '4g'
    environment:
      - SPARK_WORKER_MEMORY=4g
```

### Video Processing (FFmpeg, GStreamer)

```yaml
services:
  video-processor:
    image: jrottenberg/ffmpeg:5.1-alpine
    shm_size: '512m'   # For video frame buffering
```

## Step 3: Using tmpfs as an Alternative

Instead of setting shm_size globally, you can also use a `tmpfs` mount for `/dev/shm`:

```yaml
services:
  app:
    image: myapp:latest
    tmpfs:
      # Mount /dev/shm as tmpfs with 512 MB limit
      - /dev/shm:size=536870912,mode=1777  # 512 MB in bytes
```

This provides more control over permissions and is equivalent to `shm_size`.

## Step 4: Docker CLI Equivalent

```bash
# Docker CLI: set shared memory size
docker run -d \
  --name my-chrome \
  --shm-size=2g \
  selenium/standalone-chrome:latest

# Or use tmpfs mount:
docker run -d \
  --name my-chrome \
  --tmpfs /dev/shm:rw,size=2g \
  selenium/standalone-chrome:latest
```

## Step 5: Verify Shared Memory Size

Inside the running container:

```bash
# In the container console (Portainer Exec):
df -h /dev/shm
# Filesystem      Size  Used Avail Use% Mounted on
# shm             2.0G     0  2.0G   0% /dev/shm

# Check the size in bytes:
cat /proc/meminfo | grep Shmem

# For PostgreSQL, check configured shared_buffers:
psql -U postgres -c "SHOW shared_buffers;"
```

## Monitoring Shared Memory Usage

```bash
# On the host, check actual shared memory usage per container:
docker stats --format "table {{.Name}}\t{{.MemUsage}}" my-container

# Inside container, see what's using /dev/shm:
ls -la /dev/shm/
```

## Best Practices

- **Set shm_size explicitly** for Chrome/Selenium, ML workloads, and PostgreSQL.
- **Don't over-allocate** — shm_size counts against the container's memory limit.
- **Use `--disable-dev-shm-usage`** for Chrome if you can't increase shm_size.
- **Monitor actual usage** — set `shm_size` to match actual requirements, not arbitrary large values.

## Conclusion

Shared memory configuration in Portainer is a small but important detail that prevents mysterious crashes in applications like Chrome, PostgreSQL, and ML frameworks. By increasing `/dev/shm` from the default 64 MB to an appropriate value for your workload, you ensure these applications have the IPC resources they need to operate correctly.
