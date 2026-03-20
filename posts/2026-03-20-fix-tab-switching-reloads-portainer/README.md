# How to Fix Tab Switching Causing Long Reloads in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Performance, UI, Browser, Caching

Description: Learn how to fix excessive reload times when switching between Portainer tabs, caused by repeated API calls and lack of UI state caching.

---

Switching between Portainer tabs (Containers, Images, Volumes, Networks) triggers a fresh API call each time. On large environments or slow connections, each tab switch can take several seconds. This guide explains why it happens and how to reduce it.

## Why Tab Switching Triggers Reloads

Portainer's Angular frontend re-fetches data from the Docker API on every tab navigation. This is by design to show current state, but it can be slow when:

- The Docker host has hundreds of resources
- The Portainer server is behind a high-latency network connection
- The Docker daemon itself is slow to respond to API calls

## Step 1: Check Docker API Response Times

```bash
# Measure the raw Docker API response time for the containers endpoint
time curl -s --unix-socket /var/run/docker.sock \
  http://localhost/containers/json?all=1 | wc -c

# If this takes more than 1 second, the Docker daemon itself is slow
# Common on hosts with hundreds of stopped containers
```

## Step 2: Reduce Number of Containers

The biggest performance improvement comes from removing stopped containers that inflate API responses:

```bash
# Remove all stopped containers
docker container prune -f

# Count remaining containers
docker ps -a | wc -l
```

## Step 3: Enable Portainer Application Data Caching

For Kubernetes environments, Portainer supports application data caching. For Docker environments, this is not available but caching can be improved by:

1. Going to **Settings > General**.
2. Under **Docker environments**, enable edge scheduling cache where available.

## Step 4: Use a Faster Storage Backend

Portainer's response times depend on BoltDB query speed, which depends on disk I/O:

```bash
# Check current disk speed on the Portainer host
# Write speed
dd if=/dev/zero of=/tmp/test bs=4k count=10000 oflag=direct
# Read speed
dd if=/tmp/test of=/dev/null bs=4k count=10000
rm /tmp/test

# Target: writes >50MB/s for acceptable Portainer performance
```

Migrate the Portainer data volume to an SSD if on spinning disk.

## Step 5: Deploy Portainer Closer to the Docker Host

Network round-trip time between Portainer and the Docker daemon adds to every tab load. For remote hosts, use the Portainer Agent instead of direct API access to reduce overhead.

## Step 6: Browser-Side Optimization

Keep only one Portainer browser tab open. Multiple Portainer tabs compete for API connections and can cause browser-side congestion.

Disable browser extensions (ad blockers, privacy tools) on the Portainer URL, as they sometimes intercept and slow API calls.
