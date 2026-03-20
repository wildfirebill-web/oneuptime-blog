# How to Troubleshoot 502 Bad Gateway Errors with Nginx and Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Nginx, Troubleshooting, 502, Networking

Description: Learn how to systematically diagnose and fix 502 Bad Gateway errors when using Nginx or Nginx Proxy Manager as a reverse proxy in front of Portainer.

## Introduction

A 502 Bad Gateway error means Nginx received an invalid response from the upstream server (Portainer). This can occur due to network misconfiguration, Portainer not running, wrong port settings, or SSL/TLS mismatch between Nginx and Portainer. This guide provides a systematic approach to identifying and fixing each cause.

## Prerequisites

- Nginx or Nginx Proxy Manager deployed as a proxy for Portainer
- Access to Nginx logs and Docker commands
- Basic understanding of Docker networking

## Step 1: Verify Portainer Is Running

The most common cause of 502 is the backend simply not running:

```bash
# Check Portainer container status

docker ps | grep portainer

# If stopped, check why it stopped
docker ps -a | grep portainer
docker logs portainer --tail=50

# Restart if stopped
docker start portainer

# Check Portainer is listening on expected port
docker exec portainer netstat -tlnp 2>/dev/null || \
  docker exec portainer ss -tlnp
# Expected: 0.0.0.0:9000 (HTTP) or 0.0.0.0:9443 (HTTPS)
```

## Step 2: Verify Network Connectivity

502 often means Nginx can't reach Portainer on the network:

```bash
# Check what networks Portainer is on
docker inspect portainer | jq '.[].NetworkSettings.Networks | keys'

# Check what networks Nginx is on
docker inspect nginx-proxy-manager | jq '.[].NetworkSettings.Networks | keys'

# They must share at least one network
# If not, connect Portainer to the proxy network
docker network connect proxy portainer

# Test connectivity from Nginx to Portainer
docker exec nginx-proxy-manager wget -qO- --timeout=5 http://portainer:9000 && echo "SUCCESS" || echo "FAILED"

# Try by IP if name resolution fails
PORTAINER_IP=$(docker inspect portainer | jq -r '.[].NetworkSettings.Networks.proxy.IPAddress')
docker exec nginx-proxy-manager wget -qO- --timeout=5 "http://${PORTAINER_IP}:9000" && echo "SUCCESS" || echo "FAILED"
```

## Step 3: Check Nginx Error Logs

```bash
# For Nginx Proxy Manager
docker logs nginx-proxy-manager 2>&1 | grep -i "error\|upstream\|502\|connect"

# For standalone Nginx
docker exec nginx cat /var/log/nginx/error.log | tail -50

# For host-installed Nginx
sudo tail -50 /var/log/nginx/error.log

# Common error messages and meanings:
# "connect() failed (111: Connection refused)"
#   → Portainer not running or wrong port
# "no live upstreams while connecting to upstream"
#   → All backend servers are down/unreachable
# "upstream timed out (110: Connection timed out)"
#   → Network unreachable or firewall blocking
# "SSL_do_handshake() failed (SSL: error)"
#   → Nginx configured for HTTP but Portainer using HTTPS (or vice versa)
```

## Step 4: Fix Scheme Mismatch (HTTP vs HTTPS)

A frequent 502 cause is scheme mismatch between Nginx and Portainer:

```nginx
# WRONG: Portainer CE default is HTTP (port 9000), not HTTPS
location / {
    proxy_pass https://portainer:9000;    # Wrong! Port 9000 is HTTP
}

# CORRECT for Portainer CE default (HTTP port 9000)
location / {
    proxy_pass http://portainer:9000;
}

# CORRECT for Portainer with HTTPS enabled (port 9443)
location / {
    proxy_pass https://portainer:9443;
    proxy_ssl_verify off;    # Portainer uses self-signed cert by default
}
```

```text
# In Nginx Proxy Manager GUI:
# If Portainer uses HTTP port 9000:
#   Scheme: http, Forward Port: 9000

# If Portainer uses HTTPS port 9443:
#   Scheme: https, Forward Port: 9443
# Then add in Advanced tab: proxy_ssl_verify off;
```

## Step 5: Fix Container Name Resolution

```bash
# Verify container name is correct
docker ps --format "{{.Names}}"

# If Portainer container is named differently (e.g., portainer_portainer_1)
docker ps | grep portainer

# Use exact container name in NPM/Nginx config
# Or use the container IP address as fallback
PORTAINER_IP=$(docker inspect portainer | jq -r '.[].NetworkSettings.Networks.proxy.IPAddress')
echo "Use this IP: $PORTAINER_IP"

# In NPM: set Forward Hostname/IP to the IP instead of container name
```

## Step 6: Check for Port Conflicts

```bash
# Verify no other service is on the same published port
sudo ss -tlnp | grep ':80\|:443\|:9000'

# Check if a firewall rule is blocking
sudo iptables -L INPUT -v -n | grep DROP
sudo ufw status verbose
```

## Step 7: WebSocket Issues Causing 502

Portainer's terminal and console use WebSockets. Missing WebSocket config can cause partial 502s:

```nginx
# Required Nginx configuration for Portainer WebSockets
location / {
    proxy_pass http://portainer:9000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;     # WebSocket upgrade
    proxy_set_header Connection "upgrade";       # Must be lowercase "upgrade"
    proxy_set_header Host $host;
    proxy_read_timeout 900;                      # Long timeout for terminal sessions
}
```

```text
# In Nginx Proxy Manager:
# Details tab: "Websockets Support" checkbox must be ON
```

## Step 8: Complete Nginx Portainer Config

Working standalone Nginx configuration for reference:

```nginx
server {
    listen 443 ssl http2;
    server_name portainer.example.com;

    ssl_certificate /etc/letsencrypt/live/portainer.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portainer.example.com/privkey.pem;

    location / {
        proxy_pass http://portainer:9000;    # HTTP, not HTTPS
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 900;
    }
}
```

## Conclusion

502 Bad Gateway errors with Nginx and Portainer almost always come from one of three causes: Portainer not running, network isolation between containers, or scheme/port misconfiguration. Always check network connectivity first with `docker exec nginx wget http://portainer:9000`, then verify the scheme matches Portainer's actual protocol, and ensure WebSocket support is enabled for Portainer's console functionality.
