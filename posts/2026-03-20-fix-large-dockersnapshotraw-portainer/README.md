# How to Fix Large DockerSnapshotRaw Payloads Slowing Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Performance, Snapshot, BoltDB, Optimization

Description: Learn how to reduce oversized DockerSnapshotRaw payloads in the Portainer database that cause slow UI rendering and high memory usage during snapshot processing.

---

Portainer stores a full snapshot of each Docker environment in its BoltDB database, including all container and image metadata. When this snapshot grows very large (hundreds of containers or images with verbose labels), it causes high memory usage and slow UI rendering.

## What is DockerSnapshotRaw?

`DockerSnapshotRaw` is the raw JSON payload stored in the Portainer database for each environment snapshot. It includes the full output of:
- `GET /containers/json?all=1`
- `GET /images/json`
- `GET /networks`
- `GET /volumes`
- `GET /swarm` (if applicable)

## Step 1: Measure Snapshot Size

```bash
# Check the Portainer database size

docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db

# For more detail, use bolt CLI to inspect bucket sizes
docker run --rm -v portainer_data:/data alpine/bbolt \
  info /data/portainer.db
```

## Step 2: Remove Unused Docker Resources

The fastest way to shrink snapshots is to remove unused resources from Docker:

```bash
# Remove all unused resources in one command
# WARNING: review what this will delete first
docker system df            # See what can be freed
docker system prune -af     # Remove everything unused (including stopped containers)
```

## Step 3: Limit Container Labels

Verbose container labels inflate snapshot size. Review labels in your Compose files:

```yaml
services:
  app:
    image: myapp
    labels:
      # Only include labels that serve a purpose
      # Remove verbose metadata labels that are not needed by Portainer
      com.example.version: "1.0"
      # Avoid large multi-line labels (e.g., embedded JSON configuration)
```

## Step 4: Increase Snapshot Interval

More frequent snapshots mean more frequent large writes to BoltDB:

```bash
# Increase to 5-minute intervals to reduce write frequency
docker run -d ... portainer/portainer-ce:latest --snapshot-interval 300
```

## Step 5: Compact the Database

After cleaning up resources, compact BoltDB to reclaim space:

```bash
docker stop portainer
docker run --rm -v portainer_data:/data portainer/portainer-ce:latest --compact-db
docker start portainer
```

## Step 6: Use the --hide-label Flag

Hide containers with specific labels from Portainer's snapshot entirely:

```bash
# Start Portainer hiding containers with the "hide=true" label
docker run -d ... portainer/portainer-ce:latest \
  --hide-label hide=true
```

Then add `hide=true` as a label to non-essential containers that do not need Portainer management.
