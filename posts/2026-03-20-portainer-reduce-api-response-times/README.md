# How to Reduce API Response Times in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API Performance, Response Time, Optimization, Docker, Caching

Description: Learn how to reduce Portainer API response times by tuning snapshot intervals, adding an Nginx caching layer, and optimizing database performance.

---

Portainer's API serves the frontend UI and external integrations. Slow API responses cause sluggish UI performance and CI/CD pipeline delays. This guide covers the key causes and fixes for slow API responses.

## Measuring API Response Times

Baseline your current response times before tuning:

```bash
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"adminpassword"}' | jq -r .jwt)

# Measure key API endpoints

for endpoint in /api/endpoints /api/stacks /api/containers/json; do
  time curl -s -H "Authorization: Bearer $TOKEN" \
    "https://portainer.example.com$endpoint" > /dev/null
done
```

## Slow API Root Causes

| Symptom | Likely Cause |
|---------|-------------|
| `/api/endpoints` slow | Too many environments; large snapshots |
| `/api/stacks` slow | Large number of stacks |
| `/api/containers` slow | Large snapshot data; BoltDB I/O |
| All endpoints slow | Portainer CPU/memory pressure |

## Fix 1: Move Database to SSD

BoltDB is a file-based database with heavy I/O during snapshots. Running it on SSD dramatically improves read/write times:

```bash
# Check current disk speed under the data volume
docker exec portainer sh -c "dd if=/dev/zero of=/data/test bs=1M count=100 oflag=dsync 2>&1 | tail -1"

# If throughput is <100 MB/s, consider moving to SSD storage
```

## Fix 2: Increase Snapshot Interval

While a snapshot is being written, API reads from the database may be slower due to BoltDB's writer lock:

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 180   # Fewer writes = fewer lock conflicts
```

## Fix 3: Compact the Database

A large, fragmented BoltDB file increases read times:

```bash
docker stop portainer
docker run --rm -v portainer_data:/data \
  portainer/portainer-ce:latest --compact-db
docker start portainer
```

## Fix 4: Add Nginx Response Caching

For read-heavy API endpoints, add an Nginx cache layer:

```nginx
# Cache configuration
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=portainer_api:10m max_size=100m inactive=5m;

server {
    listen 443 ssl;
    server_name portainer.example.com;

    location /api/endpoints {
        proxy_cache portainer_api;
        proxy_cache_valid 200 60s;       # Cache for 60 seconds
        proxy_cache_key "$uri$is_args$args";
        proxy_pass http://portainer:9000;
        add_header X-Cache-Status $upstream_cache_status;
    }

    location /api/ {
        # No cache for other API calls (mutations)
        proxy_cache off;
        proxy_pass http://portainer:9000;
    }
}
```

Only cache read-only endpoints like `/api/endpoints` - never cache authentication or write operations.

## Fix 5: Allocate More CPU to Portainer

API response generation is CPU-bound when processing large snapshots. Increase the CPU allocation:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    deploy:
      resources:
        limits:
          cpus: "2.0"   # Allow up to 2 CPU cores
```

## Fix 6: Reduce BoltDB Lock Contention

Use the `--no-analytics` flag to disable anonymous usage reporting, which makes periodic API calls:

```bash
portainer/portainer-ce:latest --no-analytics
```

## Monitoring API Latency Over Time

Track API latency trends using a monitoring script:

```bash
#!/bin/bash
# monitor-api-latency.sh

TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -d '{"Username":"admin","Password":"pass"}' -H 'Content-Type: application/json' | jq -r .jwt)

while true; do
  latency=$(curl -s -w "%{time_total}" -o /dev/null \
    -H "Authorization: Bearer $TOKEN" \
    https://portainer.example.com/api/endpoints)
  echo "$(date +%H:%M:%S) /api/endpoints: ${latency}s"
  sleep 30
done
```

Feed this output into Grafana or OneUptime for trending and alerting.
