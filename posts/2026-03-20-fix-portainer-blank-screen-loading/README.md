# How to Fix Portainer Blank Screen or Loading Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Docker, Browser Cache, UI Issues

Description: Learn how to diagnose and fix Portainer blank screen or infinite loading issues caused by stale browser cache, corrupted databases, or WebSocket problems.

---

A blank screen or spinner that never resolves in Portainer is one of the most common UI complaints. The root cause is almost always one of three things: a stale browser cache, a corrupted BoltDB database, or a broken WebSocket connection. This guide covers all three.

## Step 1: Clear Browser Cache and Hard Reload

Browser cache is the most common culprit after a Portainer upgrade.

```bash
# In Chrome/Edge: Ctrl+Shift+Delete → Clear cached images and files

# Or hard reload: Ctrl+Shift+R (Windows/Linux) or Cmd+Shift+R (Mac)
```

Also try an incognito/private window to rule out extensions interfering.

## Step 2: Check Browser Console for Errors

Open browser Developer Tools (`F12`) and check the Console and Network tabs:

- `404` on `/api/websocket` → WebSocket path not forwarded by reverse proxy
- `401 Unauthorized` on `/api/auth/logout` → Expired or invalid JWT
- JavaScript errors mentioning `undefined` → Stale cached JS after upgrade

## Step 3: Check Portainer Container Logs

```bash
# Inspect Portainer server logs for startup errors
docker logs portainer --tail 100

# Look for lines like:
# level=error msg="Unable to open database"
# level=fatal msg="Failed to create JWT service"
```

## Step 4: Fix a Corrupted BoltDB Database

If logs show database errors, the BoltDB file may be corrupt:

```bash
# Stop Portainer
docker stop portainer

# Back up the existing database
docker run --rm -v portainer_data:/data alpine \
  tar czf /tmp/portainer-backup.tar.gz /data

# Remove only the corrupted database file, not your stacks
docker run --rm -v portainer_data:/data alpine \
  rm /data/portainer.db

# Restart Portainer - it will create a fresh database
docker start portainer
```

Note: This resets users and settings but leaves stack definitions intact if you have them in Git.

## Step 5: Fix WebSocket Behind a Reverse Proxy

If the blank screen occurs only behind Nginx, ensure WebSocket upgrade headers are forwarded:

```nginx
location /api/websocket {
    proxy_pass http://portainer:9000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
}
```

## Step 6: Verify Memory

A Portainer process killed by the OOM killer may restart mid-request:

```bash
# Check if Portainer was OOM-killed
dmesg | grep -i "out of memory" | grep portainer
```

If so, see the guide on fixing Portainer memory issues on low-resource hosts.
