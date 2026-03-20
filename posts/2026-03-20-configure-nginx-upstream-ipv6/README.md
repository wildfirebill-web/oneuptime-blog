# How to Configure Nginx Upstream Servers with IPv6 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Nginx, Upstream, Load Balancing, Reverse Proxy

Description: Learn how to configure Nginx upstream blocks with IPv6 server addresses for load balancing and proxying to IPv6 backends, including health checks and keepalive settings.

## Basic IPv6 Upstream Block

```nginx
upstream ipv6_backend {
    # IPv6 addresses must be wrapped in brackets
    server [2001:db8::10]:8080;
    server [2001:db8::11]:8080;
    server [2001:db8::12]:8080;
}

server {
    listen 80;
    listen [::]:80 ipv6only=on;

    server_name example.com;

    location / {
        proxy_pass http://ipv6_backend;
        proxy_set_header Host $host;
    }
}
```

## Load Balancing with IPv6 Servers

```nginx
upstream ipv6_lb {
    # Default: round-robin

    # Weighted round-robin
    server [2001:db8::10]:8080 weight=5;
    server [2001:db8::11]:8080 weight=3;
    server [2001:db8::12]:8080 weight=2;

    # Least connections
    # least_conn;

    # IP hash (sticky sessions based on client IP)
    # ip_hash;

    # Keepalive connections to backends
    keepalive 32;
}
```

## Upstream with Health Checks and Backup

```nginx
upstream ipv6_ha {
    server [2001:db8::10]:8080;
    server [2001:db8::11]:8080;

    # Backup server (only used when primary servers are down)
    server [2001:db8::12]:8080 backup;

    # Server with health check parameters
    server [2001:db8::13]:8080 max_fails=3 fail_timeout=30s;

    keepalive 16;
    keepalive_requests 100;
    keepalive_timeout 60s;
}
```

## Mixed IPv4 and IPv6 Upstream

```nginx
upstream mixed_stack {
    # IPv6 primary servers
    server [2001:db8::10]:8080 weight=3;
    server [2001:db8::11]:8080 weight=3;

    # IPv4 servers (same syntax, no brackets for IPv4)
    server 192.168.1.10:8080 weight=1;
    server 192.168.1.11:8080 weight=1;
}
```

## HTTPS Upstream to IPv6 Backends

```nginx
upstream secure_ipv6 {
    server [2001:db8::10]:443;
    server [2001:db8::11]:443;
    keepalive 16;
}

server {
    listen [::]:443 ssl ipv6only=on;
    listen 443 ssl;

    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    location / {
        proxy_pass https://secure_ipv6;
        proxy_ssl_verify     on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;
        proxy_set_header Host $host;
    }
}
```

## Dynamic Upstream with Variables

```nginx
# Resolve IPv6 backend dynamically (requires resolver)

server {
    resolver [2001:4860:4860::8888] valid=30s;
    resolver_timeout 5s;

    location / {
        set $backend "backend.internal.example.com";
        proxy_pass http://$backend:8080;
    }
}
```

## Verify Upstream Configuration

```bash
# Check nginx configuration
nginx -t

# View upstream configuration
nginx -T | grep -A 10 'upstream'

# Monitor upstream connections
ss -tn | grep ':8080'

# Check upstream health via nginx Plus API (if available)
# Open source: use access logs to track upstream responses

# View upstream server response times in logs
# Add to log_format:
# "$upstream_addr $upstream_response_time $upstream_status"
```

## Summary

Configure IPv6 upstream servers in Nginx with `server [2001:db8::10]:PORT;` - brackets are required for IPv6 addresses. Set `keepalive 32` for connection reuse. Mix IPv4 and IPv6 servers in the same upstream block. Use `weight`, `max_fails`, `fail_timeout`, and `backup` parameters for load balancing and failover. For HTTPS backends, use `proxy_pass https://upstream_name` and configure `proxy_ssl_verify`.
