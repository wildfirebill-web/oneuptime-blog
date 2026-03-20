# How to Fix Tab Switching Causing Long Reloads in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Performance, Troubleshooting, Browser, UI

Description: Fix the annoying behavior where switching between Portainer browser tabs or navigating between sections triggers full page reloads and long wait times.

## Introduction

When switching between Portainer tabs in your browser, or navigating between the Containers, Stacks, and Images sections, some users experience full page reloads that take 10-30 seconds. This is usually caused by browser tab suspension, Portainer session expiry, or the browser discarding the application state to free memory.

## Understanding Why This Happens

Portainer is a Single Page Application (SPA) built with Angular. When you switch browser tabs:

1. **Browser tab suspension**: Modern browsers suspend inactive tabs to save memory, discarding JavaScript state
2. **Session timeout**: Portainer's JWT token may expire, requiring re-authentication
3. **Angular router re-initialization**: When the app is re-activated, it re-fetches all data
4. **Large data payloads**: The re-fetch takes long because there are many containers/stacks to load

## Step 1: Disable Browser Tab Suspension

### Chrome

1. Open `chrome://flags/`
2. Search for "Automatic tab discarding"
3. Set to **Disabled**
4. Relaunch Chrome

Or use an extension like "The Great Suspender" to exclude specific tabs.

### Firefox

1. Open `about:config`
2. Search for `browser.tabs.unloadOnLowMemory`
3. Set to `false`

Also disable background tab throttling:

1. Search for `privacy.reduceTimerPrecision`
2. Set to `false`

## Step 2: Increase Portainer Session Timeout

By default, Portainer sessions expire after a period of inactivity:

```bash
# Check current session settings

# Via API: not directly configurable via API
# Via UI: Settings → Authentication → Session lifetime

# In Portainer UI:
# Settings → Authentication → Authentication settings
# Increase "Session lifetime" to a longer value (e.g., 24h or 72h)
```

## Step 3: Keep Portainer Tab Active with a Keepalive Script

Inject a keepalive that prevents the browser from suspending the Portainer tab:

```javascript
// Run in the Portainer browser console to prevent tab suspension
setInterval(() => {
  // Make a lightweight API call to keep the session alive
  fetch('/api/status')
    .then(r => console.log('Keepalive:', r.status))
    .catch(e => console.log('Keepalive failed:', e));
}, 4 * 60 * 1000);  // Every 4 minutes
```

## Step 4: Increase the Snapshot Interval

Each time you navigate to a Portainer page, it may trigger a snapshot. Increasing the interval reduces the frequency of heavy data fetches:

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
  --snapshot-interval=300
```

## Step 5: Fix Browser Memory Pressure

Tab reloads happen more frequently when the browser is under memory pressure:

```bash
# Check what else is using memory
# Close unused browser tabs
# Reduce browser extensions running on Portainer's tab

# Increase Chrome's memory limit (for advanced users)
# Start Chrome with: --max_old_space_size=8192
```

## Step 6: Optimize Portainer API Response Size

For large environments, configure Portainer to send less data:

```bash
# Use the API with filters to reduce response size
# Instead of fetching all containers, filter to specific status/stack

# Test API response sizes
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Check unfiltered response size
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints/1/docker/containers/json | wc -c

# Check filtered (running only) response size
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:9000/api/endpoints/1/docker/containers/json?filters=%7B%22status%22%3A%5B%22running%22%5D%7D" | wc -c
```

## Step 7: Enable HTTP/2 on Your Reverse Proxy

HTTP/2 multiplexing reduces the overhead of multiple API calls:

```nginx
server {
    listen 443 ssl http2;  # Enable HTTP/2
    server_name portainer.yourdomain.com;

    # ... rest of config
}
```

```bash
# Verify HTTP/2 is working
curl -I --http2 https://portainer.yourdomain.com | grep HTTP
# Should show: HTTP/2 200
```

## Step 8: Use Portainer's "Quick Actions"

Instead of navigating to full pages (which trigger full data fetches), use:

1. **Container quick actions**: Click the action buttons directly from the container list
2. **Stack actions**: Use the action menu in the stack list without opening the stack detail

These avoid triggering a full page re-render.

## Step 9: Bookmark Specific Portainer Views

Instead of navigating through multiple pages, bookmark direct URLs:

```text
# Direct URL to containers list (filtered to running)
http://portainer.yourdomain.com/#!/1/docker/containers?status=running

# Direct URL to a specific stack
http://portainer.yourdomain.com/#!/1/docker/stacks/mystack
```

## Step 10: Monitor and Tune Database Performance

Long tab-switching reload times often correlate with slow database queries:

```bash
# Enable debug mode briefly to see query times
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --log-level=DEBUG

# Check for slow operations
docker logs portainer 2>&1 | grep -iE "ms|slow|query" | head -30
```

## Conclusion

Tab switching causing long reloads in Portainer is typically a combination of browser tab suspension discarding the application state, Portainer session expiry, and large data payload re-fetches. The most effective fixes are disabling browser tab suspension for the Portainer tab, increasing the session lifetime in Portainer settings, and increasing the snapshot interval to reduce the data Portainer needs to fetch on each navigation.
