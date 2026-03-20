# How to Optimize Portainer for Large-Scale Deployments - Optimization

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Performance, Scalability, Large Deployments, Optimization, Docker

Description: Learn how to optimize Portainer for environments with hundreds of containers, multiple environments, and high API traffic by tuning snapshot intervals, database compaction, and resource allocation.

---

Portainer's performance degrades with scale when left at default settings. This guide covers the key tuning parameters for deployments with 50+ containers, 10+ environments, or high API usage.

## Performance Bottlenecks at Scale

| Bottleneck | Symptom | Fix |
|------------|---------|-----|
| Snapshot interval too short | High CPU, slow UI | Increase `--snapshot-interval` |
| BoltDB database growth | Slow page loads, high memory | Run `--compact-db` periodically |
| Too many environments active | Slow dashboard | Reduce active endpoint polling |
| Small `DockerSnapshotRaw` | Missing containers in UI | Already handled by Portainer |
| Insufficient CPU/RAM | Portainer OOM killed | Increase container resource limits |

## Tuning Snapshot Interval

The snapshot interval controls how often Portainer polls each Docker environment:

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 120    # Poll every 2 minutes instead of default 60s
```

For large environments (100+ containers, 20+ stacks), set to 300 seconds (5 minutes):

```bash
  --snapshot-interval 300
```

## Setting Resource Limits

Ensure Portainer has adequate CPU and memory for large workloads:

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    command:
      - --snapshot-interval=120
      - --log-level=warn         # Reduce logging overhead
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 256M
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - "9000:9000"
```

## Compacting the BoltDB Database

The Portainer database grows over time with snapshot data. Compact it to reclaim space and improve read/write performance:

```bash
# Stop Portainer before compacting

docker stop portainer

# Run compaction
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --data /data \
  --compact-db

# Restart Portainer
docker start portainer
```

Check database size before and after:

```bash
docker run --rm -v portainer_data:/data alpine du -sh /data/portainer.db
```

## Reducing API Polling Load

If many users or external systems are calling the Portainer API frequently, reduce unnecessary polling:

```bash
# Use API tokens with appropriate scopes - avoid admin tokens for read-only scripts
# Implement caching in external scripts that poll Portainer

# Rate-limit Portainer API via Nginx
location /api/ {
    limit_req zone=portainer_api burst=20 nodelay;
    proxy_pass http://portainer:9000;
}
```

## Deploying on SSD Storage

Portainer's BoltDB database is write-heavy during snapshots. Use SSD storage for the `portainer_data` volume:

```bash
# Create a volume on an SSD-backed mount point
docker volume create \
  --opt type=none \
  --opt device=/mnt/ssd/portainer \
  --opt o=bind \
  portainer_ssd_data
```

## Horizontal Scaling with Portainer Business

Portainer Business Edition supports cluster deployments for high availability. Configure two Portainer instances behind a load balancer sharing the same database backend for large-scale, fault-tolerant deployments.

## Monitoring Portainer's Own Performance

Use cAdvisor and Grafana to track Portainer's resource usage:

```bash
# Check Portainer CPU and memory in real time
docker stats portainer --no-stream

# Expected healthy values (adjust for your scale):
# CPU: <10% at rest, <50% during snapshots
# Memory: <512 MB for up to 50 environments
```
