# How to Set proxy_bind to a Specific IPv4 Source Address in Nginx

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, Proxy_bind, IPv4, Source IP, Outbound, Networking

Description: Use Nginx proxy_bind to control the source IPv4 address used when making outbound connections to upstream servers, enabling multi-homed routing and policy-based forwarding.

## Introduction

By default, Nginx uses the server's primary IPv4 address when connecting to upstream backends. The `proxy_bind` directive lets you select a specific source IP for outbound proxy connections-useful for multi-homed servers, policy-based routing, or backends that allowlist specific source IPs.

## Basic proxy_bind Configuration

```nginx
# /etc/nginx/conf.d/proxy-bind.conf

server {
    listen 80;
    server_name example.com;

    location / {
        # Use 10.0.0.5 as the source IP when connecting to backend
        proxy_bind 10.0.0.5;
        proxy_pass http://192.168.1.10:8080;
        proxy_set_header Host $host;
    }
}
```

## Multi-Homed Server: Different Source IPs per Location

On a server with multiple IPv4 addresses, route different traffic through different interfaces:

```nginx
server {
    listen 80;
    server_name api.example.com;

    # Internal API: use internal interface as source
    location /internal/ {
        proxy_bind 10.0.0.1;   # Internal NIC
        proxy_pass http://internal-backend:8080;
    }

    # External partner API: use public interface as source
    location /partner/ {
        proxy_bind 203.0.113.5;  # External NIC (allowlisted by partner)
        proxy_pass http://partner-api.example.net;
    }
}
```

## Using proxy_bind with Upstream Blocks

`proxy_bind` works with named upstream pools:

```nginx
upstream backend_pool {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    listen 80;

    location / {
        # All upstream connections originate from 10.0.0.5
        proxy_bind 10.0.0.5;
        proxy_pass http://backend_pool;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

## proxy_bind with Transparent Proxying

The `transparent` keyword makes Nginx bind to the original client IP (requires root and iptables):

```nginx
server {
    listen 80;

    location / {
        # Bind to the client's IP address (transparent proxy mode)
        # Requires: worker_processes running as root
        # Requires: iptables redirect rules
        proxy_bind $remote_addr transparent;
        proxy_pass http://192.168.1.10:8080;
    }
}
```

Transparent proxy requires kernel configuration:

```bash
# Allow Nginx to bind to non-local addresses

echo 1 > /proc/sys/net/ipv4/ip_nonlocal_bind

# Redirect inbound traffic to Nginx (transparent intercept)
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j TPROXY \
  --tproxy-mark 0x1/0x1 --on-port 80

# Route marked packets through loopback
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```

## Stream Module proxy_bind

`proxy_bind` also works in the `stream` block for TCP/UDP proxying:

```nginx
stream {
    server {
        listen 203.0.113.10:3306;

        # Use specific internal IP as source for MySQL connections
        proxy_bind 10.0.0.5;
        proxy_pass 10.0.0.20:3306;
    }
}
```

## Verifying Source IP

After configuring `proxy_bind`, verify the source IP seen by the backend:

```bash
# On the backend server, watch incoming connections
ss -tn sport = :8080

# The source IP column should show your proxy_bind address (e.g., 10.0.0.5)
# State    Recv-Q Send-Q  Local Address:Port  Peer Address:Port
# ESTAB    0      0       192.168.1.10:8080   10.0.0.5:54321
```

## Conclusion

`proxy_bind` gives precise control over the outbound source IPv4 address Nginx uses when reaching upstream servers. It is essential for backends with IP-based allowlists, multi-homed servers needing interface-specific routing, and transparent proxy deployments. Combine with `proxy_http_version 1.1` and keepalive for production-grade configurations.
