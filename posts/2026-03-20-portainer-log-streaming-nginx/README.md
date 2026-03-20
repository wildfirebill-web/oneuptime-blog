# How to Fix Log Streaming Issues Behind Nginx Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Nginx, Reverse Proxy, Logs, Troubleshooting

Description: Fix Portainer container log streaming failures behind Nginx, including buffering issues, timeout configuration, and WebSocket configuration for real-time log display.

## Introduction

Portainer's log streaming feature shows live container logs in the browser. Behind Nginx, logs may not update in real-time (appearing only when Nginx flushes its buffer), stop completely after a few minutes (timeout), or fail to load entirely (WebSocket issue). This guide fixes all three problems.

## Step 1: Identify the Log Streaming Problem

```bash
# Test log streaming directly (without proxy)
# Go directly to http://your-host:9000
# Open container logs
# If logs stream in real-time: proxy is the issue

# Check what protocol log streaming uses
# F12 → Network → filter by "WS" or check the logs endpoint type
# Portainer uses WebSocket for real-time log streaming
```

## Step 2: Nginx — Fix Buffering (Most Common Issue)

```nginx
server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    ssl_certificate /etc/ssl/certs/portainer.crt;
    ssl_certificate_key /etc/ssl/private/portainer.key;

    location / {
        proxy_pass https://localhost:9443;

        # HTTP/1.1 required for WebSocket and chunked transfer
        proxy_http_version 1.1;

        # WebSocket upgrade
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Standard headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # CRITICAL: Disable buffering for log streaming
        proxy_buffering off;

        # Also disable Nginx's response buffering
        proxy_cache off;

        # Disable gzip for log streaming (interferes with streaming)
        gzip off;

        # Pass X-Accel-Buffering header to disable Nginx buffering
        proxy_set_header X-Accel-Buffering no;

        # Timeouts for long log streaming sessions
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        # Large body for image uploads
        client_max_body_size 500m;
    }
}
```

## Step 3: Fix SSE (Server-Sent Events) for Log Streaming

Some Portainer versions use SSE instead of WebSocket for logs:

```nginx
# For SSE-based log endpoints
location ~ ^/api/.*/logs {
    proxy_pass https://localhost:9443;
    proxy_http_version 1.1;

    # SSE requires disabling buffering
    proxy_set_header Connection '';
    proxy_buffering off;
    proxy_cache off;

    # Required for SSE
    proxy_set_header X-Accel-Buffering no;

    # Add chunked transfer
    chunked_transfer_encoding on;

    proxy_read_timeout 3600s;

    # Headers
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

## Step 4: Fix Nginx Timeout Causing Log Interruption

```nginx
# Default Nginx settings that cause log streaming to cut off:
# proxy_read_timeout 60s (default) → logs stop after 60 seconds of no new log lines

# Increase timeouts
proxy_read_timeout 3600s;    # 1 hour
proxy_send_timeout 3600s;    # 1 hour
keepalive_timeout 3600s;     # Keep HTTP connection alive

# Also increase at the http level in nginx.conf
http {
    keepalive_timeout 3600;
    # ...
}
```

## Step 5: Check Nginx Worker Connections

```nginx
# /etc/nginx/nginx.conf
events {
    worker_connections 1024;  # Default may be too low for many concurrent connections
    # Increase if needed:
    worker_connections 4096;
}
```

## Step 6: Fix gzip Compression Interference

```nginx
# gzip can interfere with streaming responses
server {
    # Option A: Disable globally for the Portainer vhost
    gzip off;

    # Option B: Disable only for streaming content types
    location / {
        gzip_types text/plain application/json;
        # Exclude event-stream from gzip
        # gzip_proxied any;  # Remove this line
    }
}
```

## Step 7: Fix Nginx Proxy Buffer Settings

```nginx
# Disable proxy buffers entirely for log streaming
proxy_buffering off;
proxy_buffer_size 0;   # Disable when proxy_buffering is off

# Or configure smaller buffers that flush more quickly
# proxy_buffering on;
# proxy_buffer_size 4k;
# proxy_buffers 8 4k;
# proxy_busy_buffers_size 8k;
# proxy_max_temp_file_size 0;  # Don't write to temp files
```

## Step 8: Specific Log Endpoint Configuration

Create a separate location block for Portainer's log streaming endpoint:

```nginx
# Map for WebSocket upgrade detection
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    # WebSocket endpoints (console, logs, stats)
    location /api/websocket/ {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 0;       # No timeout for WebSocket
        proxy_send_timeout 0;
        proxy_buffering off;
    }

    # All other Portainer requests
    location / {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_buffering off;
    }
}
```

## Step 9: Test Log Streaming End-to-End

```bash
# First: Test log streaming directly (no proxy)
docker logs -f some-noisy-container &

# Second: Check if Portainer shows the same logs in real-time
# via the direct URL http://your-host:9000

# Third: Test via the proxy
# If #2 works but #3 doesn't: proxy is the issue
# If #2 doesn't work: Portainer or Docker issue

# Test chunked transfer encoding
curl -v -H "Authorization: Bearer $TOKEN" \
  "https://portainer.yourdomain.com/api/endpoints/1/docker/containers/CONTAINER_ID/logs?stdout=1&follow=1" 2>&1 | \
  grep -i "transfer-encoding"
# Should show: "Transfer-Encoding: chunked"
```

## Step 10: Reload Nginx After Configuration Changes

```bash
# Test configuration syntax first
sudo nginx -t

# Reload (no downtime)
sudo nginx -s reload

# Or restart (brief downtime)
sudo systemctl restart nginx

# Verify Nginx is running correctly
sudo nginx -T | grep -A 5 "portainer"
```

## Conclusion

Log streaming issues behind Nginx are almost always caused by `proxy_buffering on` (the default), which causes Nginx to accumulate log output before forwarding to the browser. The fix is `proxy_buffering off` combined with appropriate timeout settings (`proxy_read_timeout 3600s`) to prevent the proxy from closing long-lived streaming connections. For WebSocket-based log streaming, also add the `Upgrade` and `Connection: upgrade` headers.
