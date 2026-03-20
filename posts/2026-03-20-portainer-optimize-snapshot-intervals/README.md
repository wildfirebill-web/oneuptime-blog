# How to Optimize Docker Snapshot Intervals for Performance - Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Snapshot Interval, Performance, Optimization, Docker, Configuration

Description: Learn how to tune the Portainer Docker snapshot interval to balance UI freshness with system performance for your specific deployment scale.

---

The snapshot interval controls how often Portainer polls each Docker environment to refresh its internal state. The right interval depends on your deployment size, how critical real-time accuracy is, and your server's resources.

## What Snapshots Contain

Each snapshot captures the full state of a Docker environment:

- All running and stopped containers
- All images, volumes, and networks
- Container resource stats (CPU, memory, network)
- Stack states and service replica counts

Portainer stores these snapshots in its BoltDB database and serves them to the UI. The UI does not call Docker directly - it reads from the snapshot.

## Default and Recommended Intervals

```bash
# Default: 60 seconds

portainer/portainer-ce:latest
# Equivalent to:
portainer/portainer-ce:latest --snapshot-interval 60
```

| Deployment Size | Recommended Interval | Rationale |
|-----------------|---------------------|-----------|
| 1–10 containers | 30–60s | Fine; minimal overhead |
| 10–50 containers | 60–120s | Default is acceptable |
| 50–200 containers | 120–300s | Reduces database write pressure |
| 200+ containers | 300–600s | Significant resource savings |
| Edge environments | 600–1800s | Low-bandwidth, infrequent check-in |

## Setting the Snapshot Interval

Pass the flag when starting Portainer:

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 300
```

In a Docker Compose or Portainer stack:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    command:
      - --snapshot-interval=180
      - --log-level=warn
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - "9000:9000"
```

## Measuring Snapshot Overhead

Check how much time snapshots take on your system:

```bash
# Enable debug logging temporarily to see snapshot timing
docker stop portainer
docker run -d --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -p 9000:9000 \
  portainer/portainer-ce:latest \
  --log-level debug

# Watch for snapshot-related log lines
docker logs -f portainer 2>&1 | grep -i snapshot
```

Each snapshot log line shows the environment name and time taken. If snapshots take >10 seconds, increase the interval significantly.

## Impact on UI Freshness

With longer snapshot intervals, the Portainer UI may show stale data:

| Interval | Data Age at Worst Case |
|----------|----------------------|
| 30s | Up to 30 seconds stale |
| 120s | Up to 2 minutes stale |
| 300s | Up to 5 minutes stale |

For most operational uses (checking status, reviewing logs), 2–5 minute staleness is acceptable. For active deployments, check containers directly via the CLI if you need real-time status.

## Per-Environment Snapshot Control

Portainer Business Edition allows disabling snapshots for specific environments that are rarely accessed:

1. Go to **Environments > [environment name]**.
2. Under **Snapshot** settings, disable automatic polling.
3. Click **Snapshot now** manually when you need fresh data.

## Combining with Low Log Level

Reduce I/O overhead by combining a longer interval with a lower log level:

```bash
portainer/portainer-ce:latest \
  --snapshot-interval 180 \
  --log-level warn \
  --log-mode file
```

`--log-mode file` writes logs to disk instead of stdout, which reduces the load on Docker's log driver.
