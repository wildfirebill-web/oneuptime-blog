# How to Fix WebSocket Connection Issues in Portainer Behind a Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, WebSocket, Reverse Proxy, Nginx, Traefik, Networking

Description: Learn how to fix WebSocket connection failures in Portainer when deployed behind Nginx, Traefik, or Caddy, enabling the container console and log streaming features.

---

Portainer uses WebSocket connections for the container console (exec) and real-time log streaming. Standard HTTP reverse proxy configurations do not forward WebSocket upgrade requests, breaking these features.

## How WebSocket Proxying Works

A WebSocket connection starts as an HTTP request with an `Upgrade: websocket` header. The proxy must:

1. Forward the `Upgrade` and `Connection` headers
2. Use HTTP/1.1 (not HTTP/2) for the upgrade handshake
3. Not buffer the connection (streaming)

## Nginx Configuration

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name portainer.example.com;

    location / {
        proxy_pass http://portainer:9000;
        proxy_http_version 1.1;

        # Required WebSocket headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Disable buffering for WebSocket streams
        proxy_buffering off;
        proxy_cache off;

        # Keep connection alive for long-lived WebSocket sessions
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

## Traefik Configuration

```yaml
# In Traefik labels for the Portainer service
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.portainer.rule=Host(`portainer.example.com`)"
  - "traefik.http.routers.portainer.entrypoints=websecure"
  - "traefik.http.services.portainer.loadbalancer.server.port=9000"
  # Traefik handles WebSocket upgrades automatically
  # Ensure HTTP middleware is not stripping Upgrade headers
```

## Caddy Configuration

```caddy
portainer.example.com {
    reverse_proxy portainer:9000 {
        # Caddy handles WebSocket upgrade automatically
        # No extra configuration needed for basic WebSocket support
        flush_interval -1   # Disable buffering for streaming
    }
}
```

## Testing WebSocket Connectivity

```bash
# Install wscat
npm install -g wscat

# Test WebSocket connection to Portainer
wscat -c wss://portainer.example.com/api/websocket

# Should output: "Connected (press CTRL+C to quit)"
# If it errors with 400/403, the proxy is not forwarding upgrade headers
```

## Verifying in Browser DevTools

Open DevTools Network tab and filter by **WS**. When you open a container console, you should see an entry with:

- Status: `101 Switching Protocols`
- Type: `websocket`
