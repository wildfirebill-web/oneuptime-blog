# How to Configure Nginx Proxy Manager to Forward Traffic to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Nginx Proxy Manager, Proxy, Configuration, Networking

Description: Learn how to configure Nginx Proxy Manager proxy hosts to route traffic to Portainer correctly, including WebSocket support, custom headers, and troubleshooting connection issues.

## Introduction

While Nginx Proxy Manager simplifies proxy configuration through its web UI, forwarding traffic to Portainer requires specific settings to handle WebSocket connections (used by Portainer's terminal), the correct scheme, and proper header forwarding. This guide covers the complete NPM configuration for Portainer with all edge cases.

## Prerequisites

- Nginx Proxy Manager running and accessible
- Portainer running on the same Docker network as NPM
- A domain name pointing to your server

## Step 1: Verify Network Connectivity

Before configuring NPM, confirm Portainer is reachable from the NPM container:

```bash
# Get the NPM container name or ID
docker ps | grep nginx-proxy-manager

# Test connectivity from NPM to Portainer
docker exec nginx-proxy-manager wget -qO- --timeout=5 http://portainer:9000 && echo "OK" || echo "FAILED"

# If that fails, check shared networks
docker inspect nginx-proxy-manager | jq '.[].NetworkSettings.Networks | keys'
docker inspect portainer | jq '.[].NetworkSettings.Networks | keys'

# Both must share at least one network
```

## Step 2: Configure Portainer Proxy Host in NPM

In the NPM web interface at `http://YOUR_SERVER:81`:

**Details Tab:**
```
Domain Names:        portainer.example.com
Scheme:              http           (Portainer CE default port 9000 is HTTP)
Forward Hostname/IP: portainer      (container name works on shared Docker network)
Forward Port:        9000
Block Common Exploits: ON
Websockets Support:  ON             (CRITICAL — required for Portainer terminal/console)
```

**For Portainer with HTTPS (port 9443):**
```
Scheme:              https
Forward Port:        9443
```

## Step 3: SSL Configuration in NPM

**SSL Tab:**
```
SSL Certificate:     Request a new SSL Certificate
Force SSL:           ON             (redirect HTTP to HTTPS)
HTTP/2 Support:      ON
HSTS Enabled:        ON
HSTS Subdomains:     OFF            (unless you want all subdomains forced HTTPS)
Email Address:       your-email@example.com
Use a DNS Challenge: OFF            (HTTP challenge for publicly accessible server)
                     ON             (DNS challenge for private/internal server)
```

## Step 4: Advanced Tab for Custom Headers

Under the **Advanced** tab, add custom Nginx configuration for Portainer:

```nginx
# Paste in the "Custom Nginx Configuration" box in the Advanced tab
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# Increase timeouts for long-running Portainer operations
proxy_read_timeout 900;
proxy_send_timeout 900;

# WebSocket support (also ensure "Websockets Support" is ON in Details tab)
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";

# Prevent upstream from sending gzip (NPM handles it)
proxy_set_header Accept-Encoding "";
```

## Step 5: Configure Access Lists (Optional Security)

Restrict Portainer access to specific IPs via NPM Access Lists:

1. Go to **Access Lists** → **Add Access List**
2. Configure:

```
Name: Internal Only
Satisfy Any: OFF        (must match all rules)
Pass Auth to Host: OFF

Access tab:
  Action: allow
  IP/CIDR: 192.168.1.0/24    (your internal network)
  Action: allow
  IP/CIDR: 10.0.0.0/8        (VPN range)

Authorization tab:
  (optional: add HTTP basic auth as a second factor)
```

3. In the Portainer proxy host, set **Access List** to "Internal Only"

## Step 6: Verify the Configuration

```bash
# Test HTTP to HTTPS redirect
curl -I http://portainer.example.com
# Expected: 301 Location: https://portainer.example.com

# Test HTTPS connection
curl -I https://portainer.example.com
# Expected: 200 OK

# Test WebSocket upgrade header (should see Connection: upgrade in request path)
curl -I -H "Connection: Upgrade" -H "Upgrade: websocket" https://portainer.example.com
# Portainer will reject (no WS token), but Nginx should not block the upgrade headers

# Check NPM logs for any proxy errors
docker logs nginx-proxy-manager 2>&1 | grep -i "error\|portainer"
```

## Step 7: Troubleshoot NPM to Portainer Forwarding

```bash
# 502 Bad Gateway:
# → NPM can't reach Portainer
# Fix: Verify same Docker network, check portainer container is running
docker network connect proxy portainer    # Connect Portainer to NPM's network

# 504 Gateway Timeout:
# → Connection reached but Portainer didn't respond in time
# Fix: Increase proxy_read_timeout in Advanced tab

# WebSocket disconnects in Portainer console:
# → WebSocket support not enabled
# Fix: Enable "Websockets Support" in NPM proxy host Details tab

# Certificate errors:
# → If using Portainer HTTPS (port 9443), NPM needs to accept self-signed cert
# Fix: In Advanced tab, add: proxy_ssl_verify off;
```

## Conclusion

Forwarding traffic from Nginx Proxy Manager to Portainer requires enabling WebSocket support, setting the correct forward scheme and port, and configuring appropriate timeouts for long-running operations. The NPM Advanced tab's custom Nginx configuration provides fine-grained control over proxy behavior. For security, combine NPM's Access Lists with forced SSL to ensure Portainer is only accessible from trusted IP ranges over HTTPS.
