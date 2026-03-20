# How to Configure Rancher HA with NGINX - With

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Nginx, High Availability, Load Balancer, SSL, Stream Module

Description: Use NGINX as a TCP/SSL load balancer for Rancher HA with stream module configuration, upstream health checks, and Kubernetes API proxying.

## Introduction

NGINX's stream module provides TCP load balancing capabilities ideal for Rancher HA. Unlike HTTP-mode load balancing, stream mode passes TLS traffic directly to Rancher nodes (SSL passthrough), preserving end-to-end encryption.

## Prerequisites

- NGINX 1.9+ with `ngx_stream_module` (usually included in mainline builds)
- Three Rancher server nodes
- A host separate from the Rancher nodes for the load balancer

## Step 1: Verify Stream Module is Available

```bash
# Check NGINX stream module is compiled in

nginx -V 2>&1 | grep stream

# Install NGINX with stream support (Ubuntu)
apt-get install -y nginx
```

## Step 2: Configure NGINX Stream Block

```nginx
# /etc/nginx/nginx.conf

# Include main HTTP config
include /etc/nginx/conf.d/*.conf;

# Stream block for TCP load balancing (outside the http block)
stream {
    # Log format for TCP connections
    log_format stream_proxy '$remote_addr [$time_local] '
                             '$protocol $status $bytes_sent $bytes_received '
                             '$session_time "$upstream_addr"';

    access_log /var/log/nginx/stream-access.log stream_proxy;

    # Upstream group for Rancher HTTPS
    upstream rancher_servers_https {
        least_conn;
        server 10.0.0.11:443 max_fails=3 fail_timeout=5s;
        server 10.0.0.12:443 max_fails=3 fail_timeout=5s;
        server 10.0.0.13:443 max_fails=3 fail_timeout=5s;
    }

    # Upstream group for Kubernetes API
    upstream kubernetes_api {
        least_conn;
        server 10.0.0.11:6443 max_fails=3 fail_timeout=5s;
        server 10.0.0.12:6443 max_fails=3 fail_timeout=5s;
        server 10.0.0.13:6443 max_fails=3 fail_timeout=5s;
    }

    # HTTPS frontend (SSL passthrough)
    server {
        listen 443;
        proxy_pass rancher_servers_https;
        proxy_connect_timeout 4s;
        proxy_timeout 300s;    # 5 minute timeout for long-running operations
    }

    # Kubernetes API frontend
    server {
        listen 6443;
        proxy_pass kubernetes_api;
        proxy_connect_timeout 4s;
        proxy_timeout 600s;
    }
}
```

## Step 3: Configure Health Checks (NGINX Plus)

For NGINX Plus, add active health checks:

```nginx
upstream rancher_servers_https {
    least_conn;
    server 10.0.0.11:443;
    server 10.0.0.12:443;
    server 10.0.0.13:443;

    # NGINX Plus health check (not available in open-source)
    health_check interval=5s passes=2 fails=3;
}
```

For open-source NGINX, use `max_fails` and `fail_timeout` as shown above.

## Step 4: Test Configuration and Reload

```bash
# Test NGINX configuration
nginx -t

# Reload without downtime
systemctl reload nginx

# Verify stream is working
curl -k https://nginx-lb-ip/healthz
# Expected: ok
```

## Step 5: Enable Active Health Checks via TCP Module

```nginx
# Add upstream health check via upstream_conf module (if available)
upstream rancher_servers_https {
    server 10.0.0.11:443;
    server 10.0.0.12:443;
    server 10.0.0.13:443;

    # Periodic TCP health check
    health_check;
}
```

## Step 6: Monitor NGINX Status

```nginx
# Add a status endpoint for monitoring
server {
    listen 80;
    location /nginx_status {
        stub_status;
        allow 10.0.0.0/8;    # Only allow internal access
        deny all;
    }
}
```

## Conclusion

NGINX stream module provides a lightweight, high-performance TCP load balancer for Rancher HA. The `least_conn` algorithm distributes connections evenly while the `max_fails` and `fail_timeout` directives provide passive health checking for free without requiring NGINX Plus.
