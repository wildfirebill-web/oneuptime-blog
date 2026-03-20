# How to Fix Container Console Not Loading Behind a Reverse Proxy (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Console, Terminal, Reverse Proxy

Description: Resolve issues where Portainer's container console (terminal) fails to load or connect when accessed through a reverse proxy, including WebSocket configuration and HTTPS requirements.

## Introduction

Portainer's container console (the in-browser terminal) uses WebSocket over HTTPS to establish an interactive shell session. When this feature fails behind a reverse proxy, you'll see a blank terminal window, "Connecting..." that never completes, or an immediate error. This guide covers every configuration required.

## How the Container Console Works

1. User clicks "Console" in Portainer UI
2. Portainer UI opens a WebSocket connection to `/api/websocket/exec`
3. Portainer backend calls Docker's exec API
4. A bidirectional stream is established through the WebSocket
5. Each keystroke is sent through the WebSocket to the container

Any break in this chain causes the console to fail.

## Step 1: Verify the Issue Is Proxy-Related

```bash
# Test the console without the proxy (direct access)

# Open http://your-host:9000 directly (bypass the proxy)
# Try the container console

# If console works directly but not via proxy:
# → Proxy configuration is the issue

# If console fails both directly and via proxy:
# → Check Docker exec permissions and container access
```

## Step 2: Check Browser Console for Specific Errors

```text
F12 → Console tab

Common error messages:
- "WebSocket connection failed" → WebSocket not supported by proxy
- "Error 403" → CSRF or origin validation issue
- "Error 400" → WebSocket upgrade not happening
- Blank terminal, no error → Buffer/flush issue
```

## Step 3: Nginx - Console-Specific Configuration

```nginx
server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    ssl_certificate /etc/ssl/certs/portainer.crt;
    ssl_certificate_key /etc/ssl/private/portainer.key;

    # General location block
    location / {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Critical for container console
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;  # Don't buffer terminal output
    }

    # Specific location for WebSocket endpoints (console, logs, stats)
    location /api/websocket {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;

        # Mandatory WebSocket headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;

        # No timeout for interactive sessions
        proxy_read_timeout 0;
        proxy_send_timeout 0;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

## Step 4: Verify HTTPS Is Required for WebSocket

Modern browsers require WebSocket over HTTPS (`wss://`) when the page is served over HTTPS:

```bash
# If Portainer is on HTTP and proxy is on HTTPS:
# Browser cannot make wss:// connection to an http:// backend
# The proxy MUST upgrade to wss://

# Check what scheme Portainer is using
docker inspect portainer | grep -i "ssl\|tls" | head -10

# Verify proxy is using HTTPS to backend
# In Nginx: proxy_pass https://localhost:9443 (not http://)
```

## Step 5: Fix for Portainer Running on HTTP

If Portainer uses HTTP (port 9000) but proxy uses HTTPS:

```nginx
# Nginx handles TLS termination, talks HTTP to Portainer
server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    location / {
        # Connect to Portainer's HTTP port
        proxy_pass http://localhost:9000;

        proxy_http_version 1.1;

        # WebSocket upgrade
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}
```

## Step 6: Test WebSocket Handshake for Console Endpoint

```bash
# Test the exec endpoint specifically
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Test exec creation (replace CONTAINER_ID with actual container)
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "http://localhost:9000/api/endpoints/1/docker/containers/CONTAINER_ID/exec" \
  -d '{
    "AttachStdin": true,
    "AttachStdout": true,
    "AttachStderr": true,
    "Tty": true,
    "Cmd": ["/bin/sh"]
  }' | jq .Id
```

## Step 7: Fix Connection Pooling Issues

Some proxy configurations reuse connections incorrectly for WebSocket:

```nginx
# Prevent connection reuse issues for WebSocket
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    location / {
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # ...rest of config
    }
}
```

## Step 8: Fix for Kubernetes Portainer with Ingress

```yaml
# Kubernetes Ingress with WebSocket annotations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portainer-ingress
  namespace: portainer
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "500m"
    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
spec:
  rules:
    - host: portainer.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: portainer
                port:
                  number: 9000
```

## Step 9: Debugging with curl

```bash
# Simulate a WebSocket upgrade request to the console endpoint
curl -v -N \
  --http1.1 \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Authorization: Bearer $TOKEN" \
  https://portainer.yourdomain.com/api/websocket/exec?endpointId=1

# Look for: "< HTTP/1.1 101 Switching Protocols"
# Bad: "< HTTP/1.1 400 Bad Request" = proxy not upgrading
# Bad: "< HTTP/1.1 502 Bad Gateway" = backend connection failed
```

## Conclusion

Container console failures behind a reverse proxy are almost exclusively WebSocket configuration issues. The fix is ensuring your proxy sends the `Upgrade` and `Connection: upgrade` headers, sets `proxy_http_version 1.1` (required for WebSocket in Nginx), and has sufficient timeout values for interactive sessions. Additionally, ensure both the proxy and Portainer use HTTPS consistently, as browsers require `wss://` on HTTPS pages.
