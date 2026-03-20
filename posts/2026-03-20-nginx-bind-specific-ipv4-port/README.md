# How to Bind Nginx to a Specific IPv4 Address and Port

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, IPv4, Networking, Web Server, Linux

Description: Learn how to configure Nginx to bind to a specific IPv4 address and port, restricting which network interfaces Nginx listens on for incoming connections.

## Introduction

By default, Nginx listens on all available network interfaces. Binding Nginx to a specific IPv4 address and port is useful when a server has multiple network interfaces and you want to restrict which interface serves web traffic — for example, serving public traffic only on an external IP and internal traffic on a private IP.

## Basic Binding to a Specific IP

Edit your Nginx server block to specify the IP address in the `listen` directive:

```nginx
server {
    listen 192.168.1.100:80;
    server_name example.com;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

This tells Nginx to accept connections only on `192.168.1.100` port `80`, ignoring requests arriving on other interfaces.

## Binding to Multiple Specific Addresses

To serve traffic on two specific IPs (e.g., public and private):

```nginx
server {
    listen 203.0.113.10:80;   # Public IP
    listen 10.0.0.5:80;       # Private IP
    server_name example.com internal.example.com;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Binding to a Specific IP for HTTPS

```nginx
server {
    listen 203.0.113.10:443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    root /var/www/html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Internal-Only Service on a Private IP

Serve an internal dashboard only on the private network interface:

```nginx
server {
    listen 10.0.0.5:8080;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

This service is not reachable from the public internet.

## Different Configurations per Interface

```nginx
# Public-facing server block
server {
    listen 203.0.113.10:80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 203.0.113.10:443 ssl;
    server_name example.com;
    # ... SSL config and public content ...
}

# Internal admin interface on private IP
server {
    listen 10.0.0.5:8443 ssl;
    server_name admin.example.internal;
    # ... admin interface ...
}
```

## Verifying the Configuration

Test the configuration:

```bash
nginx -t
```

Reload Nginx:

```bash
systemctl reload nginx
```

Verify which ports and addresses Nginx is listening on:

```bash
ss -tlnp | grep nginx
# or
netstat -tlnp | grep nginx
```

Expected output should show:
```
LISTEN  0  511  203.0.113.10:443  0.0.0.0:*  users:(("nginx",...))
LISTEN  0  511  203.0.113.10:80   0.0.0.0:*  users:(("nginx",...))
LISTEN  0  511  10.0.0.5:8080     0.0.0.0:*  users:(("nginx",...))
```

## Using listen with default_server

If you have multiple server blocks, designate one as the default for an IP:

```nginx
server {
    listen 203.0.113.10:80 default_server;
    server_name _;
    return 444;  # Close connection for unmatched requests
}
```

## Best Practices

- Use `nginx -t` before applying any configuration change.
- Combine IP-binding with firewall rules for defense in depth.
- Bind sensitive management interfaces to loopback or private IPs only.
- Use `default_server` on your public IP to handle unmatched connections gracefully.
- Document each `listen` directive to clarify its purpose.

## Conclusion

Binding Nginx to specific IPv4 addresses gives you precise control over which network interfaces serve traffic. This is a simple but effective security practice for servers with multiple network interfaces, ensuring internal services are never accidentally exposed to the public internet.
