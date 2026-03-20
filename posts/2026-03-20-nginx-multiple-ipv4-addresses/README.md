# How to Configure Nginx with Multiple IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, IPv4, Virtual Hosting, Web Server, Network Configuration

Description: Learn how to configure Nginx to listen on multiple IPv4 addresses, enabling IP-based virtual hosting and separating traffic streams across network interfaces.

## Introduction

Servers often have multiple IPv4 addresses assigned to different network interfaces or as virtual IP addresses. Nginx can listen on multiple IPv4 addresses simultaneously, enabling IP-based virtual hosting, separation of public and private traffic, and dedicated IPs for different services.

## Assigning Multiple IPs to a Server

Before configuring Nginx, ensure multiple IPs are assigned. On Linux:

```bash
# View current IP addresses
ip addr show

# Add a secondary IP (temporary)
ip addr add 203.0.113.20/24 dev eth0

# Add permanently (varies by distro)
# Ubuntu/Debian: /etc/netplan/01-netcfg.yaml
# RHEL/CentOS: /etc/sysconfig/network-scripts/ifcfg-eth0:1
```

## Configuration 1: Different Websites on Different IPs

Host multiple websites, each on its own IP:

```nginx
# Website A on IP 203.0.113.10
server {
    listen 203.0.113.10:80;
    server_name site-a.com www.site-a.com;
    root /var/www/site-a;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

# Website B on IP 203.0.113.20
server {
    listen 203.0.113.20:80;
    server_name site-b.com www.site-b.com;
    root /var/www/site-b;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

# Website C on IP 203.0.113.30
server {
    listen 203.0.113.30:80;
    server_name site-c.com www.site-c.com;
    root /var/www/site-c;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Configuration 2: Public and Private Separation

Serve different content based on which network the request arrives on:

```nginx
# Public API - reachable from internet
server {
    listen 203.0.113.10:443 ssl;
    server_name api.example.com;

    ssl_certificate     /etc/nginx/ssl/api.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/api.example.com.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# Internal admin panel - reachable only from private network
server {
    listen 10.0.0.5:443 ssl;
    server_name admin.internal.example.com;

    ssl_certificate     /etc/nginx/ssl/admin.internal.crt;
    ssl_certificate_key /etc/nginx/ssl/admin.internal.key;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Configuration 3: Round-Robin Load Balancing Across IPs

Accept connections on multiple IPs and proxy to a backend cluster:

```nginx
upstream backend {
    server 127.0.0.1:8001 weight=1;
    server 127.0.0.1:8002 weight=1;
    server 127.0.0.1:8003 weight=1;
    keepalive 32;
}

server {
    listen 203.0.113.10:80;
    listen 203.0.113.20:80;
    listen 203.0.113.30:80;
    server_name example.com;

    location / {
        proxy_pass         http://backend;
        proxy_http_version 1.1;
        proxy_set_header   Connection "";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
    }
}
```

## Configuration 4: SSL SNI with Multiple IPs

Combine IP-based and name-based virtual hosting:

```nginx
# Default catch-all for IP 203.0.113.10
server {
    listen 203.0.113.10:443 ssl default_server;
    server_name _;
    ssl_certificate     /etc/nginx/ssl/default.crt;
    ssl_certificate_key /etc/nginx/ssl/default.key;
    return 444;
}

# Specific service on same IP using SNI
server {
    listen 203.0.113.10:443 ssl;
    server_name api.example.com;
    ssl_certificate     /etc/nginx/ssl/api.example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/api.example.com.key;
    # ...
}
```

## Verifying Multiple IP Bindings

```bash
nginx -t && systemctl reload nginx

ss -tlnp | grep nginx
# Should show one entry per IP:port combination
```

Test each IP:

```bash
curl -H "Host: site-a.com" http://203.0.113.10/
curl -H "Host: site-b.com" http://203.0.113.20/
curl -H "Host: admin.internal.example.com" https://10.0.0.5/ \
  --cacert /etc/nginx/ssl/internal-ca.crt
```

## Best Practices

- Use `default_server` on each IP's primary server block to handle unmatched requests.
- Separate SSL certificates per IP when using IP-based virtual hosting.
- Document the purpose of each IP in configuration comments.
- Use firewall rules alongside IP binding for defense in depth.
- Monitor per-IP connection counts in Nginx logs by logging `$server_addr`.

## Conclusion

Nginx's flexible `listen` directive makes it easy to handle multiple IPv4 addresses on a single server. Whether separating public from private traffic, hosting multiple sites with dedicated IPs, or accepting connections across multiple interfaces, the configuration approach is consistent and manageable.
