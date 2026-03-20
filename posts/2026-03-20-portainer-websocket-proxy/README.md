# How to Fix WebSocket Connection Issues in Portainer Behind a Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, WebSocket, Reverse Proxy, Troubleshooting, Nginx

Description: Configure reverse proxies to properly support WebSocket connections required by Portainer for container terminals, log streaming, and real-time statistics.

## Introduction

Portainer uses WebSocket connections for several key features: container console (terminal), real-time log streaming, and live container statistics. Without proper WebSocket support in your reverse proxy, these features fail with connection errors or disconnect after a few seconds.

## Features That Require WebSocket

| Feature | WebSocket Use |
|---------|--------------|
| Container Console/Terminal | Interactive shell via WebSocket |
| Log Streaming | Real-time log updates via WebSocket |
| Container Stats | Live CPU/RAM updates |
| Kubernetes Shell | `kubectl exec` sessions |

## Step 1: Diagnose WebSocket Failure

```bash
# Check browser console for WebSocket errors
# F12 → Console tab → look for:
# "WebSocket connection to 'wss://...' failed"
# "Error during WebSocket handshake"
# "Unexpected response code: 400"

# Test WebSocket support directly
# Install wscat
npm install -g wscat

wscat -c wss://portainer.yourdomain.com
# Success: "Connected (press CTRL+C to quit)"
# Failure: "error: Unexpected server response: 400/502/etc"
```

## Step 2: Nginx — Complete WebSocket Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name portainer.yourdomain.com;

    ssl_certificate /etc/ssl/certs/portainer.crt;
    ssl_certificate_key /etc/ssl/private/portainer.key;

    location / {
        proxy_pass https://localhost:9443;

        # --- WebSocket Support --- (CRITICAL)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # --- Standard Headers ---
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # --- Timeout Settings ---
        # Long timeout for terminal sessions
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_connect_timeout 60s;

        # --- Disable Buffering ---
        # Required for log streaming
        proxy_buffering off;
        proxy_cache off;

        # --- Body Size ---
        # Allow large image uploads
        client_max_body_size 500m;
    }
}
```

## Step 3: Apache — WebSocket Configuration

```apache
<VirtualHost *:443>
    ServerName portainer.yourdomain.com

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/portainer.crt
    SSLCertificateKeyFile /etc/ssl/private/portainer.key

    # Enable required modules:
    # sudo a2enmod proxy proxy_http proxy_wstunnel ssl rewrite headers

    # WebSocket requests (Upgrade header present)
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*)           wss://localhost:9443/$1 [P,L]

    # Regular HTTP requests
    ProxyPass / https://localhost:9443/
    ProxyPassReverse / https://localhost:9443/

    # SSL Verification for backend
    SSLProxyEngine on
    SSLProxyVerify none
    SSLProxyCheckPeerCN off
    SSLProxyCheckPeerName off

    # Headers
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPreserveHost On

    # Timeouts
    ProxyTimeout 3600
    Timeout 3600
</VirtualHost>
```

## Step 4: Traefik — WebSocket Configuration

Traefik supports WebSocket natively, but needs proper configuration:

```yaml
# docker-compose.yml with Traefik
version: "3.8"
services:
  traefik:
    image: traefik:v3
    command:
      - "--entrypoints.websecure.address=:443"
      - "--providers.docker=true"
      # WebSocket is enabled by default in Traefik v2+

  portainer:
    image: portainer/portainer-ce:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.yourdomain.com`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      # Backend connection
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"
      # Increase timeout for container terminals
      - "traefik.http.middlewares.portainer-compress.compress=true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
```

For Traefik static configuration:

```yaml
# traefik.yml
serversTransport:
  # Increase timeout for WebSocket connections
  respondingTimeouts:
    readTimeout: 0    # No timeout for WebSocket
    writeTimeout: 0   # No timeout for WebSocket
    idleTimeout: 3600s
```

## Step 5: HAProxy — WebSocket Configuration

```
frontend portainer_frontend
    bind *:443 ssl crt /etc/ssl/portainer.pem
    mode http
    default_backend portainer_backend

    # WebSocket detection
    acl is_websocket hdr(Upgrade) -i WebSocket
    use_backend portainer_ws_backend if is_websocket

backend portainer_backend
    mode http
    option forwardfor
    option http-server-close
    server portainer1 localhost:9443 ssl verify none

backend portainer_ws_backend
    mode http
    option forwardfor
    # Keep WebSocket connections alive
    timeout tunnel 3600s
    server portainer1 localhost:9443 ssl verify none
```

## Step 6: Test WebSocket After Fix

```bash
# Test WebSocket handshake
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  https://portainer.yourdomain.com/api/websocket/exec?endpointId=1

# Expected: HTTP 101 Switching Protocols
# Bad: HTTP 400 Bad Request (proxy not upgrading)
# Bad: HTTP 502 Bad Gateway (backend not responding)
```

## Step 7: Fix Nginx "Connection Reset" During Streaming

If log streaming works initially then cuts out:

```nginx
# Add these specific timeout settings
proxy_read_timeout 86400s;  # 24 hours for long-running sessions
proxy_send_timeout 86400s;
keepalive_timeout 86400s;

# Prevent Nginx from closing idle WebSocket connections
# in the upstream keepalive pool
upstream portainer {
    server localhost:9443;
    keepalive 10;
}
```

## Step 8: Verify with Browser Network Tab

```bash
# In Chrome DevTools → Network tab:
# 1. Filter by "WS" (WebSocket)
# 2. Connect to a container terminal in Portainer
# 3. You should see a WebSocket connection with:
#    - Status: 101 (Switching Protocols)
#    - Type: websocket
# 4. Click the connection to see Messages tab

# If you see 400 or 502: proxy is not upgrading the connection
# If connection closes immediately: proxy timeout is too short
```

## Conclusion

WebSocket support in Portainer reverse proxy configurations requires three critical settings: `proxy_http_version 1.1`, `proxy_set_header Upgrade $http_upgrade`, and `proxy_set_header Connection "upgrade"`. Without these in Nginx, or the equivalent in Apache/Traefik/HAProxy, container terminals and log streaming will fail. Increase timeouts to 1+ hours to prevent session drops during long-running terminal sessions.
