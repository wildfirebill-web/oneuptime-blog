# How to Reduce Rancher UI Load Time - Reduce Load Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, UI Performance, CDN, Browser, Optimization, User Experience

Description: Reduce Rancher UI load time by deploying behind a CDN, optimizing TLS termination, enabling compression, and tuning proxy settings.

## Introduction

Rancher's web UI is a rich single-page application (Vue.js) that fetches substantial JavaScript bundles and makes many API calls on load. Users managing dozens of clusters often experience slow UI load times. This guide covers front-end and infrastructure optimizations.

## Step 1: Add a Reverse Proxy with Compression

Place an NGINX reverse proxy in front of Rancher to serve static assets with gzip/brotli compression:

```nginx
# nginx.conf for Rancher reverse proxy

upstream rancher {
    server rancher.cattle-system.svc.cluster.local:443;
    keepalive 100;
}

server {
    listen 443 ssl http2;
    server_name rancher.example.com;

    ssl_certificate /etc/ssl/rancher/tls.crt;
    ssl_certificate_key /etc/ssl/rancher/tls.key;
    ssl_protocols TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # Enable gzip for JS/CSS assets
    gzip on;
    gzip_types application/javascript text/css application/json;
    gzip_min_length 1024;
    gzip_comp_level 6;

    # Enable Brotli if nginx-brotli module is available
    brotli on;
    brotli_types application/javascript text/css;
    brotli_comp_level 6;

    # Cache static assets (Rancher uses content-hashed filenames)
    location ~* \.(js|css|woff2|png|svg)$ {
        proxy_pass https://rancher;
        proxy_cache rancher-static-cache;
        proxy_cache_valid 200 7d;    # Cache static assets for 7 days
        add_header Cache-Control "public, max-age=604800, immutable";
    }

    location / {
        proxy_pass https://rancher;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Step 2: Configure HTTP/2

Rancher UI makes many concurrent API calls. HTTP/2 multiplexing significantly improves performance:

```bash
# Verify HTTP/2 is enabled on your load balancer
curl -I --http2 https://rancher.example.com

# HTTP/2 response should show: HTTP/2 200
```

## Step 3: Reduce API Calls on Load

Rancher UI loads cluster summaries for all clusters on the home page. Limit visible clusters for users with large numbers:

```bash
# Set a default cluster to reduce initial load
# Users can configure this in their profile:
# User Avatar > Preferences > Landing Page > Specific Cluster
```

## Step 4: Use a CDN for Global Teams

For distributed teams, put a CDN (Cloudflare, CloudFront) in front of Rancher:

```yaml
# Cloudflare Configuration
# - Enable Brotli compression
# - Set Cache Rules for /assets/* paths with TTL 7 days
# - Enable HTTP/2 and HTTP/3 (QUIC)
# - Enable Argo Smart Routing for API calls
# - Set SSL mode to Full (Strict)
```

## Step 5: Optimize Browser Settings

Advise users to:

1. Use Chrome or Edge for best performance (V8 JS engine)
2. Ensure browser caching is enabled (not incognito mode for regular use)
3. Use the Rancher CLI for bulk operations instead of the UI

## Step 6: Monitor UI Performance

```bash
# Use Chrome DevTools to measure load time
# Open DevTools > Network > Disable cache > Hard reload
# Look at:
# - DOMContentLoaded time (should be < 3s)
# - Bundle sizes (main JS bundle is often the bottleneck)
# - API call waterfall (look for sequential blocking calls)
```

## Conclusion

The biggest UI load time improvements come from HTTP/2 multiplexing (eliminates request queuing), compression (reduces bundle transfer size by 60-70%), and static asset caching (eliminates repeat downloads). For global teams, a CDN reduces latency for users far from the Rancher server's data center.
