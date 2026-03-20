# How to Configure Nginx Upstream Servers with IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Nginx, IPv4, Upstream, Load Balancing, Reverse Proxy

Description: Configure Nginx upstream server groups with IPv4 addresses for load balancing, including round-robin, least connections, and IP hash methods.

## Introduction

Nginx `upstream` blocks define groups of backend servers for proxying and load balancing. Using explicit IPv4 addresses in upstream definitions ensures deterministic routing and avoids DNS resolution of hostnames to IPv6 addresses.

## Basic Upstream with IPv4 Servers

```nginx
# /etc/nginx/nginx.conf

http {
    upstream backend {
        server 10.0.0.10:8080;
        server 10.0.0.11:8080;
        server 10.0.0.12:8080;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

## Weighted Load Balancing

```nginx
upstream backend {
    # Server 10.0.0.10 gets 3x more requests than the others
    server 10.0.0.10:8080 weight=3;
    server 10.0.0.11:8080 weight=1;
    server 10.0.0.12:8080 weight=1;
}
```

## Least Connections Method

```nginx
upstream backend {
    # Send requests to the server with fewest active connections
    least_conn;

    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}
```

## IP Hash (Sticky Sessions)

```nginx
upstream backend {
    # Client IP always goes to the same backend
    ip_hash;

    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}
```

## Mark a Server as Backup

```nginx
upstream backend {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
    # backup is only used when all others are unavailable
    server 10.0.0.12:8080 backup;
}
```

## Health Checks and Failure Thresholds

```nginx
upstream backend {
    server 10.0.0.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:8080 max_fails=3 fail_timeout=30s;
}
```

## Upstream on Specific Ports

```nginx
upstream api_servers {
    server 10.0.0.10:3000;
    server 10.0.0.11:3000;
}

upstream static_servers {
    server 10.0.0.20:8080;
    server 10.0.0.21:8080;
}
```

## Test the Configuration

```bash
# Verify syntax
nginx -t

# Reload
systemctl reload nginx

# Test load balancing
for i in {1..10}; do
    curl -s http://example.com/health | head -1
done
```

## Conclusion

Nginx upstream blocks define backend server pools using explicit IPv4 addresses. Choose `round-robin` (default), `least_conn`, or `ip_hash` for load balancing method. Use `weight`, `backup`, `max_fails`, and `fail_timeout` to control server behavior. Explicit IPv4 addresses avoid DNS resolution issues that could route traffic to IPv6 backends.
