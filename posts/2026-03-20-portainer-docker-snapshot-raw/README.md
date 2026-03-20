# How to Fix Large DockerSnapshotRaw Payloads Slowing Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, Troubleshooting, Database Optimization

Description: Reduce oversized DockerSnapshotRaw payloads stored in Portainer's database that cause slow API responses, large database files, and UI performance degradation.

## Introduction

Portainer stores Docker environment snapshots in its BoltDB database as `DockerSnapshotRaw` entries. In environments with many containers, networks, volumes, and images, these snapshots can become very large - sometimes hundreds of megabytes - causing slow API responses, large database files, and UI sluggishness. This guide explains how to manage and reduce these payloads.

## What Is DockerSnapshotRaw?

Every time Portainer takes a snapshot of a Docker environment, it serializes the entire Docker state (all containers, images, volumes, networks, and their inspect data) and stores it in BoltDB. The more resources you have, the larger this payload becomes.

## Step 1: Identify the Database Size

```bash
# Check the size of Portainer's database

docker run --rm \
  -v portainer_data:/data \
  alpine ls -lh /data/portainer.db

# Check total volume size
docker volume inspect portainer_data
MOUNTPOINT=$(docker volume inspect portainer_data --format '{{.Mountpoint}}')
du -sh $MOUNTPOINT
```

## Step 2: Check Snapshot Data Size via API

```bash
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Check the snapshot raw data size for an endpoint
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/1/docker/snapshot | \
  wc -c  # Count bytes

# Large environments can return 1-50MB per snapshot
```

## Step 3: Increase Snapshot Interval

Reducing snapshot frequency means fewer large payloads stored over time:

```bash
docker stop portainer && docker rm portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --snapshot-interval=600   # 10 minutes - reduces write frequency
```

## Step 4: Compact the Database

BoltDB doesn't reclaim free pages automatically. Compact it regularly:

```bash
# Stop Portainer
docker stop portainer && docker rm portainer

# Run compact-db flag
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Check size before/after
docker run --rm \
  -v portainer_data:/data \
  alpine ls -lh /data/portainer.db

# Restart Portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 5: Clean Up Docker Resources

The snapshot size is directly proportional to the number of resources Docker has. Cleaning up reduces payload size:

```bash
# Remove all stopped containers
docker container prune -f

# Remove unused images (saves snapshot size significantly)
docker image prune -a -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Check how much space was freed
docker system df
```

## Step 6: Remove Old Image Versions

Old, untagged images are included in snapshots:

```bash
# List dangling images (untagged)
docker images -f "dangling=true"

# Remove dangling images
docker rmi $(docker images -q -f "dangling=true")

# Or with prune
docker image prune

# Check total image count
docker images | wc -l
```

## Step 7: Archive or Remove Unnecessary Environments

Each additional environment has its own snapshot payload:

```bash
# Check how many environments Portainer is managing
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | jq 'length'

# List all environments with their last activity
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | \
  jq '.[] | {id: .Id, name: .Name, status: .Status}'

# Remove stale/unused environments via UI or API
curl -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/ENDPOINT_ID
```

## Step 8: Schedule Regular Cleanup

```bash
#!/bin/bash
# portainer-maintenance.sh
# Schedule with: 0 2 * * 0  (every Sunday at 2 AM)

echo "$(date): Starting Portainer maintenance"

# Clean Docker resources to reduce snapshot size
docker container prune -f
docker image prune -f
docker volume prune -f
docker network prune -f

echo "$(date): Docker cleanup complete"
echo "$(date): Current Docker disk usage:"
docker system df

# Compact Portainer database
echo "$(date): Compacting Portainer database..."
docker stop portainer
sleep 5

docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Restart Portainer
docker start portainer

echo "$(date): Maintenance complete"
```

## Step 9: Move to Agent-Based Snapshots

Instead of the Portainer server pulling full snapshots, use the Agent:

The Agent takes snapshots locally and sends a summary to the server, reducing network payload:

```bash
# Deploy the Agent on the Docker host
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# In Portainer, change the environment to use Agent URL
# instead of direct Docker socket
# Go to: Environments → Edit → URL: tcp://host:9001
```

## Step 10: Monitor Snapshot Size Over Time

```bash
#!/bin/bash
# monitor-snapshot-size.sh
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

ENDPOINTS=$(curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | jq -r '.[].Id')

for EP in $ENDPOINTS; do
  EP_NAME=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/endpoints/$EP" | jq -r '.Name')

  SIZE=$(curl -s -H "Authorization: Bearer $TOKEN" \
    "http://localhost:9000/api/endpoints/$EP/docker/snapshot" | wc -c)

  echo "Environment: $EP_NAME | Snapshot size: ${SIZE} bytes"
done
```

## Conclusion

Large `DockerSnapshotRaw` payloads are a natural consequence of managing large Docker environments in Portainer. The most impactful fixes are: increasing the snapshot interval to reduce write frequency, regularly cleaning unused Docker resources to reduce snapshot size, running `--compact-db` periodically to reclaim BoltDB free pages, and using the Agent mode which uses more efficient snapshot transmission.
