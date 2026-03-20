# How to Configure Nginx to Listen Only on IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Nginx, IPv4, Web Server, Networking, Configuration

Description: Configure Nginx to listen only on IPv4 addresses, preventing it from binding to IPv6 sockets, and ensuring all traffic is handled over IPv4.

## Introduction

By default, Nginx may listen on both IPv4 and IPv6. To restrict Nginx to IPv4 only, specify IPv4 addresses in `listen` directives and remove any IPv6 `listen` directives. This is useful in environments where IPv6 is not needed or not available.

## Basic IPv4-Only listen Directive

```nginx
# /etc/nginx/sites-available/default
server {
    # Listen on all IPv4 addresses, port 80
    listen 80;

    # Do NOT add: listen [::]:80;
    # The absence of IPv6 listen directives = IPv4 only

    server_name example.com;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

## Explicit IPv4 Binding

```nginx
server {
    # Explicitly bind to 0.0.0.0 (all IPv4 interfaces)
    listen 0.0.0.0:80;

    server_name example.com;

    location / {
        root /var/www/html;
    }
}
```

## Disable IPv6 in nginx.conf

```nginx
# /etc/nginx/nginx.conf
http {
    server {
        # IPv4 only — no [::] entries
        listen 80;
        listen 443 ssl;
        server_name _;

        # ... rest of config
    }
}
```

## Verify Nginx is Not Listening on IPv6

```bash
# Apply config
nginx -t
systemctl reload nginx

# Check listening sockets — should show only 0.0.0.0:80
ss -tlnp | grep nginx

# Expected:
# LISTEN  0  511  0.0.0.0:80   0.0.0.0:*  users:(("nginx",...))
# No [::]:80 entries
```

## Disable IPv6 System-Wide for Nginx

```bash
# If IPv6 is disabled system-wide, Nginx cannot bind to IPv6 anyway
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
```

## IPv4-Only HTTPS Configuration

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.pem;
    ssl_certificate_key /etc/ssl/private/example.key;

    location / {
        root /var/www/html;
    }
}
```

## Test and Reload

```bash
# Test configuration syntax
nginx -t

# Reload to apply changes
systemctl reload nginx

# Verify
curl -v http://example.com
```

## Conclusion

Nginx listens on IPv4 only when `listen` directives use IPv4 addresses (like `listen 80;` or `listen 0.0.0.0:80;`) without any `listen [::]:80;` IPv6 directives. Remove or omit all `[::]:` entries from your server blocks. Verify with `ss -tlnp | grep nginx`.
