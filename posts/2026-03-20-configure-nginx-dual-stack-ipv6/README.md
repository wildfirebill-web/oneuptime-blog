# How to Configure Nginx Dual-Stack (IPv4 and IPv6) Listeners

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Nginx, Dual-Stack, Web Server, Network Configuration

Description: Learn how to configure Nginx to listen on both IPv4 and IPv6 simultaneously (dual-stack mode), including the correct use of the ipv6only parameter.

## Dual-Stack Configuration Basics

```nginx
# The key to dual-stack in Nginx is using separate listen directives

# for IPv4 and IPv6

server {
    # IPv4 on all interfaces
    listen 0.0.0.0:80;

    # IPv6 on all interfaces (ipv6only=on prevents IPv4 overlap)
    listen [::]:80 ipv6only=on;

    server_name example.com;

    location / {
        root /var/www/html;
    }
}
```

## Understanding ipv6only Parameter

```nginx
# ipv6only=on (recommended for dual-stack):
# [::]:80 accepts ONLY IPv6 connections
# 0.0.0.0:80 accepts ONLY IPv4 connections
# Both listeners can coexist on the same port

# ipv6only=off (or omitted) on Linux:
# [::]:80 may accept BOTH IPv4 and IPv6 connections
# (depends on net.ipv6.bindv6only kernel setting)
# This can cause "Address already in use" errors

# Best practice: Always use ipv6only=on for explicit dual-stack
```

## Full Dual-Stack HTTPS Configuration

```nginx
server {
    # Redirect HTTP to HTTPS (both IPv4 and IPv6)
    listen 80;
    listen [::]:80 ipv6only=on;

    server_name example.com;

    return 301 https://$host$request_uri;
}

server {
    # HTTPS dual-stack
    listen 443 ssl http2;
    listen [::]:443 ssl http2 ipv6only=on;

    server_name example.com;

    ssl_certificate     /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

## Multiple Virtual Hosts with Dual-Stack

```nginx
# Server block 1: example.com
server {
    listen 80;
    listen [::]:80 ipv6only=on;
    server_name example.com;
    # ...
}

# Server block 2: api.example.com
server {
    listen 80;
    listen [::]:80;   # Note: no ipv6only on subsequent blocks (it's per-socket, not per-server)
    server_name api.example.com;
    # ...
}
```

Note: `ipv6only` is a socket-level option. Only the first server block's listen directive sets it for a given port. Subsequent server blocks sharing the same `[::]:80` socket inherit the setting.

## Check the ipv6only Linux Kernel Default

```bash
# Check Linux default for IPv6-only binding
cat /proc/sys/net/ipv6/bindv6only

# 0 = IPv6 sockets can accept IPv4 (IPv4-mapped)
# 1 = IPv6 sockets only accept IPv6

# If bindv6only=1, [::]:80 only accepts IPv6 regardless of ipv6only setting
# In this case, you MUST use separate listen 0.0.0.0:80 for IPv4
```

## Verify Dual-Stack is Working

```bash
# Check Nginx is listening on both
ss -tlnp | grep nginx
# Should show: *:80 (IPv4) and [::]:80 (IPv6)

# Test IPv4
curl -4 http://example.com

# Test IPv6
curl -6 http://example.com

# Test from specific IPv6 address
curl --interface 2001:db8::10 http://example.com

# Check logs for both IPv4 and IPv6 client addresses
tail -f /var/log/nginx/access.log
```

## Summary

Configure Nginx dual-stack with `listen 0.0.0.0:80;` and `listen [::]:80 ipv6only=on;` in the same server block. The `ipv6only=on` parameter ensures each listener only handles its respective IP version. For HTTPS, use the same pattern with port 443. Verify with `ss -tlnp | grep nginx` and test with `curl -4` and `curl -6`.
