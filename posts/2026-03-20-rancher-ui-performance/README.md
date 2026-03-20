# How to Reduce Rancher UI Load Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, UI Performance, Nginx, CDN, Optimization

Description: Reduce Rancher UI load time through CDN configuration, browser caching, compression, and resource optimization techniques for improved user experience.

## Introduction

A slow Rancher UI frustrates operators managing many clusters. UI performance depends on network latency to the Rancher server, asset delivery speed, and browser caching. This guide covers practical techniques to improve Rancher UI responsiveness for distributed teams.

## Prerequisites

- Rancher installation accessible via HTTPS
- NGINX or another reverse proxy in front of Rancher
- CDN (optional but recommended for geographically distributed teams)

## Step 1: Configure NGINX Reverse Proxy with Caching

```nginx
# nginx.conf - Optimized Rancher reverse proxy
upstream rancher_servers {
    least_conn;
    server rancher-01:443;
    server rancher-02:443;
    server rancher-03:443;

    keepalive 32;  # Persistent connections
}

server {
    listen 443 ssl http2;
    server_name rancher.example.com;

    ssl_certificate     /etc/ssl/certs/rancher.crt;
    ssl_certificate_key /etc/ssl/private/rancher.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # Enable compression
    gzip on;
    gzip_vary on;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        image/svg+xml
        font/woff2;
    gzip_min_length 1024;
    gzip_comp_level 6;

    # Cache static assets
    location ~* \.(js|css|png|jpg|ico|svg|woff2|woff|ttf)$ {
        proxy_pass https://rancher_servers;
        proxy_cache rancher_cache;
        proxy_cache_valid 200 7d;
        proxy_cache_use_stale error timeout updating;
        add_header Cache-Control "public, max-age=604800, immutable";
        add_header X-Cache-Status $upstream_cache_status;
    }

    # Proxy all other requests
    location / {
        proxy_pass https://rancher_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 900;
    }
}

# Proxy cache zone
proxy_cache_path /var/cache/nginx/rancher
    levels=1:2
    keys_zone=rancher_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;
```

## Step 2: Configure CDN for Rancher UI Assets

```bash
# For CloudFront distribution in front of Rancher

# Create behavior for static assets
aws cloudfront create-distribution \
  --distribution-config '{
    "Origins": {
      "Items": [{
        "Id": "rancher-origin",
        "DomainName": "rancher.example.com",
        "CustomOriginConfig": {
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only"
        }
      }]
    },
    "CacheBehaviors": {
      "Items": [{
        "PathPattern": "*.js",
        "TargetOriginId": "rancher-origin",
        "CachePolicyId": "cache-optimized-policy-id",
        "ViewerProtocolPolicy": "https-only",
        "Compress": true
      }]
    }
  }'
```

## Step 3: Enable HTTP/2 Push for Critical Resources

```nginx
# Push critical Rancher UI assets
location = / {
    http2_push /dashboard/assets/index.js;
    http2_push /dashboard/assets/index.css;
    proxy_pass https://rancher_servers;
}
```

## Step 4: Configure Rancher UI Feature Flags

```bash
# Disable unused features to reduce UI complexity
# In Rancher settings

# Enable only necessary features
curl -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"value": "istio=false,legacy=false"}' \
  "https://rancher.example.com/v3/settings/feature-gates"

# Disable legacy UI (Rancher v2.6+)
curl -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"value": "false"}' \
  "https://rancher.example.com/v3/settings/ui-legacy"
```

## Step 5: Optimize API Response Times

```bash
# Rancher UI makes many API calls on load
# Check slow API endpoints

kubectl logs -n cattle-system deployment/rancher | \
  grep "slow request" | \
  awk '{print $NF}' | \
  sort | uniq -c | sort -rn | head -20

# Enable API pagination to reduce payload size
# Rancher uses pagination by default, but check limits
curl -H "Authorization: Bearer $TOKEN" \
  "https://rancher.example.com/v3/clusters?limit=100&marker=0"
```

## Step 6: Browser Cache Configuration

```bash
# Ensure proper cache headers are set for Rancher assets
curl -I https://rancher.example.com/dashboard/assets/index.js

# Expected headers:
# Cache-Control: public, max-age=31536000, immutable
# Etag: "version-hash"
# Content-Encoding: gzip

# If cache headers are missing, add them in your reverse proxy
location ~* \.(js|css)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
    expires 1y;
}
```

## Step 7: Monitor UI Performance

```bash
# Use Lighthouse or WebPageTest to measure UI performance
npx lighthouse https://rancher.example.com \
  --chrome-flags="--headless" \
  --output=json \
  --output-path=rancher-perf.json

# Extract key metrics
cat rancher-perf.json | jq '.categories.performance.score,
  .audits["first-contentful-paint"].displayValue,
  .audits["time-to-interactive"].displayValue'
```

## Conclusion

Rancher UI performance improvement requires a multi-layered approach. NGINX caching with HTTP/2, gzip compression, and proper cache headers for static assets can cut initial load time by 40-60%. For globally distributed teams, a CDN in front of Rancher delivers assets from edge locations near each user. Combined with disabling unused Rancher features and monitoring API response times, these optimizations create a significantly more responsive management experience.
