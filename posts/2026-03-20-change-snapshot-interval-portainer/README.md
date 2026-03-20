# How to Change the Snapshot Interval in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Configuration, Performance, Snapshot, Docker

Description: Learn how to adjust Portainer's environment snapshot interval to balance data freshness with system performance.

---

Portainer periodically polls connected environments to update the state displayed in the UI - this is called a snapshot. By default, snapshots occur every 5 minutes. On busy or resource-constrained systems, you may want to adjust this interval.

## What is a Snapshot?

A snapshot is a point-in-time capture of an environment's state, including:
- Running and stopped containers
- Images, volumes, and networks
- Stack status

Snapshots are used to display environment data in the Portainer UI without querying the Docker API on every page load. More frequent snapshots mean fresher data but higher CPU and memory usage.

## Change Snapshot Interval via the UI

1. Log in to Portainer as an administrator
2. Navigate to **Settings**
3. Under **Application settings**, find **Snapshot interval**
4. Enter the new interval in seconds (e.g., `300` for 5 minutes, `60` for 1 minute)
5. Click **Save application settings**

## Change Snapshot Interval via the CLI Flag

You can also set the snapshot interval at container startup:

```bash
# Start Portainer with a custom snapshot interval (60 seconds)

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval 60
```

## Change Snapshot Interval via API

```bash
# Authenticate
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Set snapshot interval to 120 seconds
curl -X PUT \
  https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"SnapshotInterval": "120s"}' \
  --insecure
```

## Recommended Intervals by Use Case

| Scenario | Recommended Interval | Reasoning |
|----------|---------------------|-----------|
| Development/testing | 30–60 seconds | Fast feedback for active development |
| Production (standard) | 300 seconds (5 min) | Good balance of freshness and performance |
| Production (large clusters) | 600–900 seconds | Reduce load on environments with many nodes |
| Edge environments (async) | 900–3600 seconds | Edge agents have limited bandwidth |

## Impact of Very Short Intervals

Setting the snapshot interval too low (under 30 seconds) on large environments can cause:
- Increased CPU on the Portainer server
- Higher API call load on Docker daemons
- Potential network congestion for remote agents
- Slower UI response times

## Disable Snapshots for Specific Environments

In Portainer BE, you can disable snapshots per environment from **Environments > Edit Environment > Advanced settings**.

---

*Monitor your container environments with [OneUptime](https://oneuptime.com) for real-time uptime and performance tracking.*
