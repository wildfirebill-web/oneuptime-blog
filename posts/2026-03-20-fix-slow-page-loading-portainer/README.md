# How to Fix Slow Page Loading with Many Resources in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Performance, UI, Large Deployments, Optimization

Description: Learn how to fix slow Portainer page loading when managing hundreds of containers, images, or volumes by optimizing snapshot settings and filtering resources.

---

Portainer's UI can become sluggish when managing hosts with hundreds of containers or large numbers of images and volumes. The bottleneck is usually the snapshot payload size and the frontend rendering time.

## Step 1: Profile the Slow Page

Open browser DevTools (F12) and go to the Network tab. Reload the slow page:

- Large `GET /api/endpoints/<id>/docker/containers/json` responses (>1MB) indicate too many containers in the snapshot
- Slow `GET /api/endpoints/<id>/docker/images/json` responses indicate image list is massive
- JS CPU spikes in the Performance tab indicate frontend rendering is the bottleneck

## Step 2: Clean Up Unused Resources

The fastest fix is reducing the number of resources Portainer must manage:

```bash
# Remove stopped containers
docker container prune -f

# Remove dangling images (untagged intermediate layers)
docker image prune -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Or do all at once (caution in production)
docker system prune -f
```

## Step 3: Increase Snapshot Interval

Reducing snapshot frequency decreases server-side processing:

```bash
# Restart Portainer with 5-minute intervals
docker run -d ... portainer/portainer-ce:latest --snapshot-interval 300
```

## Step 4: Use Portainer Resource Filtering

In the Portainer UI, use filters and search to limit displayed resources:

- Use the **Search** bar to filter by container name
- Use **Status** filters (Running/Stopped) to hide irrelevant containers
- Use **Stack** filter to view only one stack at a time

## Step 5: Paginate Large Lists

Portainer Business Edition supports configurable pagination. In **Settings > General**, reduce the default page size for container lists on hosts with hundreds of containers.

## Step 6: Disable Auto-Refresh

In the container list, disable automatic refresh if you do not need real-time updates:

The refresh interval can be controlled in the Portainer UI settings. Disabling polling reduces background API calls that slow down page interactions.

## Step 7: Upgrade Hardware

For environments with 500+ containers, Portainer benefits from:

- SSD storage for the BoltDB database (critical for snapshot writes)
- At least 2GB RAM dedicated to Portainer
- Fast network between Portainer server and agents
