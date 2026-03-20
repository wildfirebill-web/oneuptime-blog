# How to Set Up Nginx Stream Module for TCP/UDP Proxying on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Stream Module, TCP, UDP, IPv4, Proxy, Layer 4

Description: Configure the Nginx stream module to proxy TCP and UDP traffic on specific IPv4 addresses, enabling Layer 4 load balancing for databases, game servers, and other non-HTTP services.

## Introduction

The `ngx_stream_core_module` extends Nginx beyond HTTP to proxy arbitrary TCP and UDP streams. This is ideal for load balancing MySQL, PostgreSQL, Redis, MQTT, DNS, and other protocols over IPv4.

## Prerequisites

- Nginx compiled with `--with-stream` (included in `nginx-full` package or mainline builds)
- Root/sudo access to configure low-numbered ports

Verify stream support:
```bash
nginx -V 2>&1 | grep -o with-stream
# Should output: with-stream
```

## Basic TCP Proxy Configuration

The `stream` block is separate from the `http` block in `nginx.conf`:

```nginx
# /etc/nginx/nginx.conf

events {
    worker_connections 1024;
}

# HTTP block (your existing config)
http {
    include /etc/nginx/conf.d/*.conf;
}

# Stream block for TCP/UDP proxying
stream {
    include /etc/nginx/stream.d/*.conf;
}
```

```nginx
# /etc/nginx/stream.d/mysql-proxy.conf

upstream mysql_backends {
    server 10.0.0.10:3306;
    server 10.0.0.11:3306;
}

server {
    # Listen on specific IPv4 address for MySQL connections
    listen 203.0.113.10:3306;

    proxy_pass mysql_backends;

    # Connection timeouts
    proxy_connect_timeout 5s;
    proxy_timeout 60s;
}
```

## UDP Load Balancing (DNS Example)

```nginx
# /etc/nginx/stream.d/dns-proxy.conf

upstream dns_servers {
    server 8.8.8.8:53;
    server 1.1.1.1:53;
}

server {
    # UDP listener on specific IPv4
    listen 203.0.113.10:53 udp;

    proxy_pass dns_servers;
    proxy_timeout 3s;
    proxy_responses 1;  # Expected number of UDP responses per request
}
```

## TCP Load Balancing with Health Checks

```nginx
stream {
    upstream redis_backends {
        server 10.0.0.10:6379 max_fails=3 fail_timeout=30s;
        server 10.0.0.11:6379 max_fails=3 fail_timeout=30s;
        server 10.0.0.12:6379 backup;
    }

    server {
        listen 203.0.113.10:6379;
        proxy_pass redis_backends;
        proxy_connect_timeout 3s;
        proxy_timeout 120s;

        # Pass real client IP via PROXY protocol
        proxy_protocol on;
    }
}
```

## SSL/TLS Termination for TCP Streams

```nginx
stream {
    server {
        listen 203.0.113.10:5432 ssl;

        ssl_certificate     /etc/ssl/certs/server.crt;
        ssl_certificate_key /etc/ssl/private/server.key;
        ssl_protocols TLSv1.2 TLSv1.3;

        # Proxy to plain PostgreSQL backends
        proxy_pass 10.0.0.10:5432;
    }
}
```

## Access Control in Stream Blocks

```nginx
stream {
    # Access control module for stream
    server {
        listen 203.0.113.10:3306;

        # Only allow internal subnet
        allow 10.0.0.0/8;
        deny all;

        proxy_pass mysql_backends;
    }
}
```

## Verifying Stream Proxy

```bash
# Test TCP proxy for MySQL
mysql -h 203.0.113.10 -P 3306 -u testuser -p

# Test UDP DNS proxy
dig @203.0.113.10 example.com

# Check stream connections
ss -tn dst 203.0.113.10:3306
```

## Conclusion

The Nginx stream module brings the same powerful proxy and load balancing capabilities of the HTTP module to Layer 4 TCP and UDP traffic. Configure it in a separate `stream {}` block, use the same `upstream` syntax for backends, and combine with SSL termination or PROXY protocol for production deployments.
