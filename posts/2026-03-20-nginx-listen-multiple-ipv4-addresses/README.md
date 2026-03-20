# How to Set Up Nginx to Listen on Multiple IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Nginx, IPv4, Web Server, Multi-Homed, Networking

Description: Configure Nginx to listen on multiple IPv4 addresses simultaneously, serving different content or redirecting traffic based on the destination IP address.

## Introduction

A server with multiple IPv4 addresses (multi-homed) can run Nginx with different configurations per IP. Use multiple `listen` directives or separate `server` blocks to handle each IP differently. This enables virtual hosting by IP, where each address serves a different website or application.

## Multiple listen Directives in One Block

```nginx
# Same content served on two different IPs
server {
    listen 192.168.1.100:80;
    listen 10.0.0.10:80;

    server_name _;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

## Separate Server Blocks Per IP

```nginx
# Public website
server {
    listen 203.0.113.1:80;
    server_name www.example.com;

    location / {
        root /var/www/public;
    }
}

# API on different IP
server {
    listen 203.0.113.2:80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}

# Admin on internal IP
server {
    listen 10.0.0.10:80;
    server_name admin.example.com;

    # Restrict to internal network
    allow 10.0.0.0/8;
    deny all;

    location / {
        root /var/www/admin;
    }
}
```

## Listen on All IPv4 Plus Specific

```nginx
# Catch-all for any IPv4
server {
    listen 0.0.0.0:80 default_server;
    server_name _;
    return 444;  # Close connection silently
}

# Specific IP gets real content
server {
    listen 203.0.113.1:80;
    server_name example.com;
    location / {
        root /var/www/html;
    }
}
```

## Verify Multiple Bindings

```bash
# Test configuration
nginx -t

# Reload
systemctl reload nginx

# Show all listening sockets for Nginx
ss -4tlnp | grep nginx

# Expected:
# LISTEN  0  511  203.0.113.1:80   0.0.0.0:*  ...
# LISTEN  0  511  10.0.0.10:80     0.0.0.0:*  ...
```

## Test Each Binding

```bash
# Test each IP independently
curl -H "Host: www.example.com" http://203.0.113.1/
curl -H "Host: admin.example.com" http://10.0.0.10/
```

## Use Nginx Map for IP-Based Logic

```nginx
# nginx.conf http block
map $server_addr $site_root {
    203.0.113.1  /var/www/site1;
    203.0.113.2  /var/www/site2;
    default      /var/www/default;
}

server {
    listen 0.0.0.0:80;
    root $site_root;
}
```

## Conclusion

Nginx listens on multiple IPv4 addresses by adding multiple `listen <ip>:port` directives. Use separate `server` blocks for different content per IP. Verify all bindings with `ss -4tlnp | grep nginx`. The `default_server` flag on a `listen` directive designates the catch-all for unmatched requests on that IP.
