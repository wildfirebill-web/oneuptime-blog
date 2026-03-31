# How to Configure Redis with NGINX TCP Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, NGINX, TCP Proxy, Load Balancer, Networking, Configuration

Description: Configure NGINX as a TCP stream proxy for Redis using the stream module, enabling load balancing, connection limiting, and health checks for Redis deployments.

---

## NGINX Stream Module for TCP Proxying

NGINX can proxy TCP connections (not just HTTP) using the `ngx_stream_core_module`. This allows NGINX to act as a TCP load balancer and proxy for Redis.

NGINX stream proxying is a simple TCP passthrough - it does not understand the Redis protocol (unlike HAProxy with its Redis-specific health checks). However, it works well for basic load balancing and connection proxying.

## Installing NGINX with Stream Support

The `stream` module is included in NGINX from version 1.9.0 and is available in most standard packages:

```bash
sudo apt update
sudo apt install -y nginx

# Verify stream module is available
nginx -V 2>&1 | grep stream
# Should include: --with-stream
```

If stream is not included, install `nginx-full`:

```bash
sudo apt install -y nginx-full
```

## Basic Redis TCP Proxy Configuration

The `stream` block goes at the top level of `nginx.conf`, not inside `http`:

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
}

# HTTP block (optional, keep if you have web services)
# http { ... }

# Stream block for TCP proxying
stream {
    upstream redis_backend {
        server 10.0.0.1:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_backend;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
    }
}
```

## Load Balancing Across Multiple Redis Nodes

NGINX supports several load balancing methods in the stream module:

```nginx
stream {
    # Round-robin (default)
    upstream redis_replicas {
        server 10.0.0.3:6379;
        server 10.0.0.4:6379;
    }

    # Least connections
    upstream redis_read_lb {
        least_conn;
        server 10.0.0.3:6379;
        server 10.0.0.4:6379;
    }

    # IP hash for session persistence
    upstream redis_sticky {
        hash $remote_addr consistent;
        server 10.0.0.3:6379;
        server 10.0.0.4:6379;
    }

    server {
        listen 6380;
        proxy_pass redis_replicas;
    }
}
```

## Separate Ports for Reads and Writes

```nginx
stream {
    # Primary (write endpoint)
    upstream redis_primary {
        server 10.0.0.1:6379;
        server 10.0.0.2:6379 backup;
    }

    # Replicas (read endpoint)
    upstream redis_replicas {
        least_conn;
        server 10.0.0.3:6379;
        server 10.0.0.4:6379;
        server 10.0.0.5:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_primary;
        proxy_connect_timeout 1s;
        proxy_timeout 10s;
    }

    server {
        listen 6380;
        proxy_pass redis_replicas;
        proxy_connect_timeout 1s;
        proxy_timeout 10s;
    }
}
```

## Health Checks (NGINX Plus Only)

Active health checks for TCP streams are only available in NGINX Plus. For open-source NGINX, use passive health checks:

```nginx
stream {
    upstream redis_backend {
        server 10.0.0.1:6379 max_fails=3 fail_timeout=30s;
        server 10.0.0.2:6379 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 6379;
        proxy_pass redis_backend;
        proxy_connect_timeout 2s;
        proxy_timeout 30s;
    }
}
```

With passive health checks:
- `max_fails=3` - mark the server as down after 3 failed connections
- `fail_timeout=30s` - time window for counting failures and time to wait before retrying a down server

## NGINX Plus Active Health Checks (if available)

```nginx
stream {
    upstream redis_backend {
        zone redis 64k;
        server 10.0.0.1:6379;
        server 10.0.0.2:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_backend;

        health_check interval=5s fails=2 passes=1;
    }
}
```

## Connection Limits

Protect Redis from connection flooding:

```nginx
stream {
    # Limit connections per IP
    limit_conn_zone $remote_addr zone=redis_conn:10m;

    upstream redis_backend {
        server 10.0.0.1:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_backend;
        limit_conn redis_conn 10;  # Max 10 connections per IP
    }
}
```

## TLS Termination for Redis

NGINX can terminate TLS before proxying to a plain-text Redis:

```nginx
stream {
    upstream redis_plain {
        server 127.0.0.1:6379;
    }

    server {
        listen 6380 ssl;
        proxy_pass redis_plain;

        ssl_certificate     /etc/nginx/ssl/redis.crt;
        ssl_certificate_key /etc/nginx/ssl/redis.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        proxy_connect_timeout 1s;
        proxy_timeout 3s;
    }
}
```

Clients connect to NGINX on port 6380 with TLS. NGINX proxies to Redis on port 6379 without TLS.

## Logging Stream Connections

```nginx
stream {
    log_format redis_log '$remote_addr [$time_local] '
                         '$protocol $status $bytes_sent $bytes_received '
                         '$session_time "$upstream_addr"';

    access_log /var/log/nginx/redis_access.log redis_log;

    upstream redis_backend {
        server 10.0.0.1:6379;
    }

    server {
        listen 6379;
        proxy_pass redis_backend;
    }
}
```

## Testing and Reloading

```bash
# Validate configuration
sudo nginx -t

# Reload without downtime
sudo systemctl reload nginx

# Test Redis through NGINX
redis-cli -h localhost -p 6379 -a yourpassword PING

# Check NGINX is listening
ss -tlnp | grep nginx
```

## Summary

NGINX proxies Redis TCP connections via the `stream` module with round-robin, least-connections, or consistent hash load balancing. Configure separate listen ports for write (primary) and read (replica) traffic, use passive health checks with `max_fails` to detect and bypass failed nodes, and optionally add TLS termination so Redis can run without TLS while clients connect securely. For active health checks and advanced health monitoring, upgrade to NGINX Plus.
