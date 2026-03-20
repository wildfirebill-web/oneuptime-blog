# How to Fix 'Origin Invalid' Errors After Upgrading Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, CSRF, Origin, Security, Upgrade, Reverse Proxy

Description: Learn how to fix 'Origin Invalid' or 'CSRF check failed' errors that appear after upgrading Portainer, caused by stricter origin validation introduced in newer versions.

---

Newer versions of Portainer enforce stricter CSRF origin validation. After upgrading, users behind reverse proxies or accessing Portainer via non-standard URLs may see "Origin Invalid" errors when trying to log in or perform actions.

## Why This Happens After Upgrade

Portainer added cross-origin protection in its API. The server compares the `Origin` header in requests against the expected server URL. If they don't match - typically because a proxy rewrites URLs or because you're accessing Portainer via IP when it expects a hostname - all POST requests fail.

## Step 1: Identify the Mismatch

```bash
# Check what origin Portainer is receiving

docker logs portainer 2>&1 | grep -i "origin\|csrf\|referer" | tail -20

# You'll see something like:
# "origin header value invalid: expected https://portainer.example.com, got http://192.168.1.50:9000"
```

## Step 2: Access Portainer via Its Configured URL

The simplest fix: access Portainer via the URL that matches its configuration:

```bash
# If Portainer is behind a reverse proxy at https://portainer.example.com,
# always access it via that URL, not via direct IP:port
```

## Step 3: Set X-Forwarded-Host Header in Reverse Proxy

Configure your reverse proxy to send the correct host header:

```nginx
location / {
    proxy_pass http://portainer:9000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # This header helps Portainer validate origins correctly
    proxy_set_header X-Forwarded-Host $host;
}
```

## Step 4: Set --base-url Flag

If Portainer is served from a subpath (e.g., `/portainer`), set the base URL flag:

```bash
docker run -d ... portainer/portainer-ce:latest \
  --base-url /portainer
```

Without this, Portainer generates incorrect redirect URLs and origin checks fail.

## Step 5: Roll Back to Previous Version Temporarily

If you need immediate access while diagnosing:

```bash
# Run the previous version with the same data
docker stop portainer && docker rm portainer
docker run -d --name portainer \
  -v portainer_data:/data \
  portainer/portainer-ce:2.29.3  # Use your previous version
```

## Step 6: Allow Direct IP Access

For homelab use where you always access Portainer via IP:

```bash
# Portainer accepts the HTTPS_PROXY or similar environment variable
# The simplest approach is to not use a reverse proxy for the admin interface
# and access via: http://<host-ip>:9000
```

Ensure your browser address bar URL matches exactly how Portainer was configured.
