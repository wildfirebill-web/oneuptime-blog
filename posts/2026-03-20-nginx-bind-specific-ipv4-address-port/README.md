# How to Bind Nginx to a Specific IPv4 Address and Port

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Nginx, IPv4, Web Server, Networking, Configuration

Description: Bind Nginx to a specific IPv4 address and port, enabling different server configurations for different IPs on a multi-homed server.

## Introduction

Binding Nginx to a specific IP address is useful on multi-homed servers where different services should be accessible only on specific interfaces. Use the `listen <ip>:<port>` syntax in server blocks to restrict Nginx to specific IP-port combinations.

## Bind to a Specific IP and Port

```nginx
# /etc/nginx/sites-available/app1

server {
    # Only accept connections to 192.168.1.100 on port 80
    listen 192.168.1.100:80;

    server_name app1.example.com;

    location / {
        root /var/www/app1;
        index index.html;
    }
}
```

## Multiple Server Blocks on Different IPs

```nginx
# Public-facing server
server {
    listen 203.0.113.1:80;
    server_name public.example.com;

    location / {
        root /var/www/public;
    }
}

# Internal management server (private IP only)
server {
    listen 10.0.0.10:8080;
    server_name admin.example.com;

    location / {
        root /var/www/admin;
    }
}
```

## Bind HTTPS to Specific IP

```nginx
server {
    listen 192.168.1.100:443 ssl;
    server_name secure.example.com;

    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    location / {
        root /var/www/secure;
    }
}
```

## Verify Binding

```bash
# Test configuration
nginx -t

# Reload
systemctl reload nginx

# Verify Nginx is bound to the specific IP:port
ss -tlnp | grep nginx

# Expected:
# LISTEN  0  511  192.168.1.100:80   0.0.0.0:*  users:(("nginx",...))
```

## Test with curl

```bash
# Test the specific IP binding
curl -v http://192.168.1.100/

# This should fail (different IP not bound)
curl -v http://192.168.1.200/
# Returns connection refused or no response
```

## Loopback-Only Binding (Internal Services)

```nginx
server {
    # Only accessible from localhost
    listen 127.0.0.1:8080;
    server_name localhost;

    location / {
        proxy_pass http://backend;
    }
}
```

## Default Server for an IP

```nginx
server {
    # default_server means this block catches unmatched server_names
    listen 192.168.1.100:80 default_server;
    server_name _;

    return 404;
}
```

## Conclusion

Use `listen <ip>:<port>` to bind Nginx to a specific IPv4 address. This allows different server blocks to serve different IPs on the same machine. Verify with `ss -tlnp | grep nginx` to confirm the exact IP:port binding. The `default_server` flag designates the catch-all block for a given IP:port combination.
