# How to Configure Nginx Reverse Proxy with IPv6 Backends

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Nginx, Reverse Proxy, proxy_pass, Backend

Description: Learn how to configure Nginx as a reverse proxy to forward requests to IPv6 backend servers, including proper address formatting and health checks.

## Proxy to an IPv6 Backend

```nginx
server {
    listen 80;
    listen [::]:80 ipv6only=on;

    server_name example.com;

    location / {
        # IPv6 backend address must be enclosed in brackets
        proxy_pass http://[2001:db8::10]:8080;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Proxy to a Specific IPv6 Address

```nginx
server {
    listen [::]:443 ssl ipv6only=on;
    listen 443 ssl;

    server_name api.example.com;

    ssl_certificate     /etc/ssl/certs/api.example.com.crt;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;

    location /api/ {
        proxy_pass http://[2001:db8::20]:3000/;

        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 10s;
        proxy_read_timeout    60s;
        proxy_send_timeout    60s;
    }
}
```

## Using Upstream Block with IPv6

```nginx
upstream backend_ipv6 {
    # IPv6 backends - addresses in brackets
    server [2001:db8::10]:8080;
    server [2001:db8::11]:8080;
    server [2001:db8::12]:8080;

    keepalive 32;
}

server {
    listen 80;
    listen [::]:80 ipv6only=on;

    server_name example.com;

    location / {
        proxy_pass http://backend_ipv6;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # Enable keepalive
    }
}
```

## Mixed IPv4/IPv6 Upstream

```nginx
upstream mixed_backend {
    # Mix of IPv4 and IPv6 backends
    server [2001:db8::10]:8080 weight=3;
    server [2001:db8::11]:8080 weight=3;
    server 192.168.1.10:8080   weight=1;  # IPv4 fallback

    keepalive 16;
}
```

## Preserve Client IPv6 Address

```nginx
server {
    listen [::]:80 ipv6only=on;

    location / {
        proxy_pass http://[2001:db8::10]:8080;

        # Pass original client IP (works for both IPv4 and IPv6 clients)
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # For IPv6 clients, $remote_addr will be e.g. 2001:db8::20
    }
}
```

## Test Reverse Proxy to IPv6 Backend

```bash
# Test that Nginx can reach the IPv6 backend
curl -6 http://[2001:db8::10]:8080/health

# Test through Nginx
curl -6 http://example.com/

# Check Nginx is connecting to the IPv6 backend
nginx -T | grep proxy_pass | grep '\['

# Check error logs for connection failures
tail -f /var/log/nginx/error.log
```

## Summary

Configure Nginx reverse proxy to IPv6 backends by enclosing the IPv6 address in brackets in `proxy_pass`: `proxy_pass http://[2001:db8::10]:8080;`. In `upstream` blocks, use `server [2001:db8::10]:8080;`. Always set `proxy_set_header X-Real-IP $remote_addr` to preserve client IP. For HTTPS backends, use `proxy_pass https://[2001:db8::10]:443`. IPv6 addresses in nginx configuration always require brackets.
