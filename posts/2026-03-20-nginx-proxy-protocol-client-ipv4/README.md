# How to Set Up Nginx PROXY Protocol to Preserve IPv4 Client Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, PROXY Protocol, IPv4, Client IP, Load Balancer, HAProxy

Description: Configure Nginx to receive and forward the PROXY Protocol header from upstream load balancers to preserve the original client IPv4 address in access logs and application headers.

## Introduction

When a load balancer (HAProxy, AWS NLB, etc.) proxies traffic to Nginx, the client IP is replaced by the load balancer's IP. PROXY Protocol is a TCP-level header prepended to connections that carries the original client's IP and port. Nginx can receive PROXY Protocol from upstream and forward it via X-Forwarded-For headers to backend applications.

## Architecture

```
Client (203.0.113.10)
    ↓ TCP connection
HAProxy/NLB (10.0.1.1)
    ↓ PROXY Protocol v1 header + TCP to Nginx
Nginx (10.0.1.10)
    → Logs 203.0.113.10 (real client IP)
    → Passes X-Real-IP: 203.0.113.10 to app
```

## Configuring Nginx to Accept PROXY Protocol

```nginx
# /etc/nginx/sites-available/app.conf

server {
    # Accept PROXY Protocol on port 80
    listen 80 proxy_protocol;

    # Optionally accept on 443 too
    listen 443 ssl proxy_protocol;

    ssl_certificate     /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    # Restrict PROXY Protocol to trusted upstream IPs only
    set_real_ip_from 10.0.1.0/24;    # Load balancer subnet
    real_ip_header   proxy_protocol;

    server_name app.example.com;

    location / {
        proxy_pass http://backend;

        # Forward the real client IP to the application
        proxy_set_header X-Real-IP       $proxy_protocol_addr;
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        proxy_set_header Host            $host;
    }
}
```

## Update Access Log to Show Real Client IP

```nginx
http {
    # Custom log format using $proxy_protocol_addr
    log_format main_pp '$proxy_protocol_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main_pp;
}
```

## PROXY Protocol in the Stream Module (TCP)

For TCP load balancing (Nginx stream module):

```nginx
stream {
    upstream backend-tcp {
        server 10.0.2.10:8080;
    }

    server {
        listen 8080 proxy_protocol;
        proxy_pass backend-tcp;

        # Accept PROXY Protocol from trusted upstream
        set_real_ip_from 10.0.1.0/24;
    }
}
```

## Configuring HAProxy to Send PROXY Protocol

On the upstream HAProxy load balancer:

```
backend nginx-backends
    server nginx1 10.0.1.10:80 send-proxy         # PROXY Protocol v1
    server nginx2 10.0.1.11:80 send-proxy-v2      # PROXY Protocol v2
```

## Configuring AWS NLB for PROXY Protocol

```bash
# Enable PROXY Protocol v2 on an NLB target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=proxy_protocol_v2.enabled,Value=true
```

## Validating PROXY Protocol Reception

```bash
# Check access log shows real client IP (not load balancer IP)
sudo tail -f /var/log/nginx/access.log

# Verify the variable is populated
# Add this temporarily to a location block:
# return 200 "$proxy_protocol_addr\n";
curl http://app.example.com/

# Should return: 203.0.113.10 (your real IP)
```

## Important: Only Accept from Trusted Upstream

```nginx
# CRITICAL: Only accept PROXY Protocol from your load balancer subnets
# Never expose proxy_protocol on ports accessible to untrusted clients
# A client could spoof their IP by sending a fake PROXY Protocol header

set_real_ip_from 10.0.1.1/32;   # Only your specific load balancer
real_ip_header proxy_protocol;
```

## Conclusion

Add `proxy_protocol` to the `listen` directive to enable reception. Use `set_real_ip_from` to restrict which upstream IPs are trusted to send PROXY Protocol — otherwise clients can spoof their IPs. Set `$proxy_protocol_addr` in headers passed to backend applications. Configure the corresponding `send-proxy` or `send-proxy-v2` on the HAProxy backend, or enable it on AWS NLB target groups.
