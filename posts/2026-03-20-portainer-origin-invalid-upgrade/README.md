# How to Fix 'Origin Invalid' Errors After Upgrading Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Security, Upgrade, CORS

Description: Resolve 'Origin Invalid' or CORS-related errors that appear after upgrading Portainer, caused by stricter origin validation introduced in recent versions.

## Introduction

Starting with Portainer 2.19+, stricter origin validation was introduced as a security measure to prevent CSRF attacks. If you access Portainer via an IP address, a different hostname than expected, or behind a proxy that doesn't set the correct headers, you'll see "Invalid Origin" errors. This guide explains the root cause and all available fixes.

## Why Origin Validation Was Added

Without origin validation, a malicious website could make cross-origin requests to your Portainer instance using your browser's session cookies. Portainer now validates that the `Origin` header in requests matches the expected server URL.

## Step 1: Identify the Error

```bash
# Check browser console for the error

# Open F12 → Console
# Error typically looks like:
# "Invalid origin"
# "CORS policy: No 'Access-Control-Allow-Origin' header"

# Check Portainer logs
docker logs portainer 2>&1 | grep -i "origin\|cors\|invalid" | tail -20
```

## Step 2: Access via the Correct URL

The most reliable fix is to access Portainer via a consistent, trusted URL:

```bash
# After the upgrade, if you're getting "Invalid Origin":
# 1. Clear browser cache (Ctrl+Shift+Delete)
# 2. Navigate to the HTTPS URL: https://portainer.yourdomain.com
# 3. Do NOT mix IP access and hostname access

# The issue often occurs when:
# - You upgraded and now access via HTTPS but old cookies are from HTTP
# - You're accessing via IP but Portainer is configured with a domain
```

## Step 3: Fix Origin Validation via --tunnel-addr Flag

Portainer uses the `--tunnel-addr` value to determine valid origins. If your URL doesn't match:

```bash
# This is NOT about Edge tunnels - when behind a reverse proxy
# you need to tell Portainer its external URL

# Unfortunately, Portainer doesn't have a direct --external-url flag
# The workaround is to ensure your reverse proxy sends the correct Host header

# Test what origin Portainer receives
docker logs portainer 2>&1 | grep "Origin" | head -5
```

## Step 4: Fix Reverse Proxy Configuration

The most common cause is a reverse proxy not forwarding the correct `Host` and `Origin` headers:

### Nginx

```nginx
server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    location / {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;

        # CRITICAL: Forward host header correctly
        proxy_set_header Host $host;
        proxy_set_header Origin $http_origin;

        # WebSocket headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Real IP headers
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Traefik

```yaml
# Traefik middleware to handle headers
http:
  middlewares:
    portainer-headers:
      headers:
        customRequestHeaders:
          X-Forwarded-Proto: "https"
        # Don't strip the Origin header
        accessControlAllowOriginList:
          - "https://portainer.yourdomain.com"
```

### Caddy

```text
portainer.yourdomain.com {
    reverse_proxy localhost:9443 {
        transport http {
            tls
            tls_insecure_skip_verify
        }
        # Caddy automatically forwards Host header
    }
}
```

## Step 5: Downgrade and Migrate

If origin validation is breaking your specific setup:

```bash
# Temporarily downgrade to a pre-2.19 version while investigating
docker stop portainer && docker rm portainer

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.18.4

# Then investigate the proper proxy configuration
# before upgrading again
```

## Step 6: Fix Docker Compose + Traefik Origin Issues

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.yourdomain.com`)"
      - "traefik.http.routers.portainer.tls=true"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"
      - "traefik.http.services.portainer.loadbalancer.server.scheme=http"
      # Pass HTTPS indicator to Portainer
      - "traefik.http.middlewares.portainer-https.headers.customrequestheaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.portainer.middlewares=portainer-https"
```

## Step 7: Clear All Portainer-Related Cookies

After fixing the reverse proxy, clear all Portainer cookies:

```javascript
// In browser console on Portainer page:
// Delete all cookies for this domain
document.cookie.split(";").forEach(function(c) {
  document.cookie = c.replace(/^ +/, "").replace(/=.*/, "=;expires=" + new Date().toUTCString() + ";path=/");
});

// Clear localStorage
localStorage.clear();

// Reload
location.reload(true);
```

## Step 8: Use HTTPS Consistently

After upgrading, always use HTTPS:

```bash
# Redirect HTTP to HTTPS in Nginx
server {
    listen 80;
    server_name portainer.yourdomain.com;
    return 301 https://$host$request_uri;
}

# Make Portainer HTTPS-only
docker run -d \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --http-disabled
```

## Conclusion

"Origin Invalid" errors after upgrading Portainer are caused by the stricter CSRF protection added in v2.19+. The fix is almost always ensuring your reverse proxy correctly forwards the `Host` header and the `X-Forwarded-Proto: https` header, then clearing your browser cookies. Access Portainer via a consistent domain/URL rather than mixing IP and hostname access.
