# How to Configure Layer 7 IPv6 Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Layer 7, Load Balancing, HAProxy, Nginx, HTTP, Application

Description: A guide to configuring Layer 7 IPv6 load balancing with HAProxy and nginx, enabling content-based routing and application-aware traffic distribution over IPv6.

Layer 7 load balancing distributes traffic based on HTTP headers, URL paths, cookies, and other application-layer content. HAProxy and nginx both provide excellent Layer 7 IPv6 load balancing with full header inspection capabilities.

## HAProxy Layer 7 IPv6 Load Balancing

```text
# /etc/haproxy/haproxy.cfg

global
    daemon
    maxconn 100000
    log /dev/log local0

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    option httplog
    option forwardfor    # Add X-Forwarded-For header

###############################
# Frontend: Accept IPv6 clients

###############################

frontend ipv6_http
    bind [::]:80 v6only
    bind 0.0.0.0:80

    # Content-based routing
    acl is_api path_beg /api/
    acl is_static path_beg /static/
    acl is_health path /health

    use_backend api_servers if is_api
    use_backend static_servers if is_static
    use_backend health_backend if is_health
    default_backend web_servers

frontend ipv6_https
    bind [::]:443 v6only ssl crt /etc/ssl/certs/combined.pem
    bind 0.0.0.0:443 ssl crt /etc/ssl/certs/combined.pem

    # Forward IPv6 client address
    http-request set-header X-Real-IPv6 %[src] if { src_is_ipv6 }

    default_backend web_servers

###############################
# Backends
###############################

backend web_servers
    balance leastconn
    cookie SERVER insert indirect nocache    # Session persistence by cookie

    option httpchk GET /health HTTP/1.1
    http-check expect status 200

    server web1 [2001:db8::web1]:8080 check cookie web1
    server web2 [2001:db8::web2]:8080 check cookie web2
    server web3 [2001:db8::web3]:8080 check cookie web3

backend api_servers
    balance roundrobin
    option httpchk GET /api/health HTTP/1.1
    http-check expect status 200

    server api1 [2001:db8::api1]:8080 check
    server api2 [2001:db8::api2]:8080 check

backend static_servers
    balance uri    # Hash by URI for cache efficiency
    server cdn1 [2001:db8::cdn1]:8080 check
    server cdn2 [2001:db8::cdn2]:8080 check

backend health_backend
    balance roundrobin
    server health1 [2001:db8::health]:8080 check
```

## nginx Layer 7 IPv6 Load Balancing

```nginx
# /etc/nginx/nginx.conf

upstream ipv6_backends {
    # IPv6 upstream servers with weighted round-robin (default)
    server [2001:db8::server1]:8080 weight=3;
    server [2001:db8::server2]:8080 weight=2;
    server [2001:db8::server3]:8080 weight=1;

    # Use one load balancing method: least_conn for active connection balancing
    least_conn;

    keepalive 32;    # Keep persistent connections to backends
}

server {
    # Listen on both IPv4 and IPv6
    listen 80;
    listen [::]:80;

    listen 443 ssl;
    listen [::]:443 ssl;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    server_name example.com;

    # Content-based routing
    location /api/ {
        proxy_pass http://ipv6_api_backends;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # Preserve IPv6 client address
        proxy_set_header X-Original-IPv6 $remote_addr;
    }

    location /static/ {
        proxy_pass http://ipv6_static_backends;
        proxy_cache_valid 200 1d;
    }

    location / {
        proxy_pass http://ipv6_backends;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

upstream ipv6_api_backends {
    server [2001:db8::api1]:8081;
    server [2001:db8::api2]:8081;
    least_conn;
}
```

## IPv6 Client IP Preservation

Layer 7 load balancers break the direct connection, so the original client IP must be forwarded in headers:

```nginx
# nginx: capture original IPv6 client address
real_ip_header X-Forwarded-For;
set_real_ip_from 2001:db8:lb::/64;    # Trust your LB subnet
real_ip_recursive on;
```

```text
# HAProxy: extract IPv6 client for logging
log-format "%ci:%cp [%t] %ft %b/%s %Tq/%Tw/%Tc/%Tr/%Tt %ST %B %hr %hs %{+Q}r"
# %ci = client IP (IPv4 or IPv6)
```

## Testing Layer 7 IPv6 Load Balancing

```bash
# Test HTTP routing
curl -6 http://example.com/
curl -6 http://example.com/api/users
curl -6 http://example.com/static/image.png

# Test with specific IPv6 source
curl --interface "2001:db8::client" http://example.com/

# Check HAProxy stats
curl http://localhost:8404/stats | grep -E "server|backend"

# Check nginx status
curl http://localhost/nginx_status
```

Layer 7 IPv6 load balancing with HAProxy or nginx enables sophisticated routing decisions based on application content, making it the right choice for modern microservice architectures where different IPv6 clients need to be routed to appropriate service tiers.
