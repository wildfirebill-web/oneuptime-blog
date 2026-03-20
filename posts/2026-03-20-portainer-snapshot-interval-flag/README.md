# How to Use the --snapshot-interval Flag in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, CLI Flags, Performance, Snapshot, Configuration, Docker

Description: Learn how to use the --snapshot-interval flag to control how frequently Portainer polls Docker environments for state changes, balancing freshness with resource usage.

---

Portainer polls each registered Docker environment every `--snapshot-interval` seconds to update the in-memory state used by the UI. The default is 60 seconds. This guide explains when and how to change it.

## Default Behavior

```bash
# Default: poll all environments every 60 seconds

docker run -d portainer/portainer-ce:latest
# Equivalent to:
docker run -d portainer/portainer-ce:latest --snapshot-interval 60
```

## When to Increase the Interval

Increase the interval to reduce load in these scenarios:

- Hosts with hundreds of containers (large snapshot payloads)
- Low-RAM hosts where snapshot processing causes OOM kills
- High-latency connections between Portainer and Docker hosts
- Environments that do not change frequently

```bash
# Poll every 5 minutes (300 seconds)
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 300
```

## When to Decrease the Interval

Decrease the interval if you need near-real-time state in the UI:

```bash
# Poll every 15 seconds (more responsive UI)
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 15
```

Note: Very short intervals increase CPU and memory usage on both the Portainer server and all connected Docker hosts.

## Effect on UI Data Freshness

The snapshot interval directly affects how stale the UI data can be:

| Interval | Maximum UI Staleness | Use Case |
|---|---|---|
| 15s | 15 seconds | Active development environments |
| 60s | 1 minute | Default - most use cases |
| 300s | 5 minutes | Low-resource hosts |
| 3600s | 1 hour | Read-mostly monitoring dashboards |

## Docker Compose Configuration

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    command:
      - --snapshot-interval=120
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

## Force a Manual Snapshot Refresh

For immediate UI refresh without waiting for the interval, restart Portainer. It immediately snapshots all environments on startup regardless of the configured interval.
