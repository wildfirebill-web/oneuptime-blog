# How to Fix 502 Bad Gateway with Cloudflare Tunnel in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloudflare, Troubleshooting, 502, Tunnel

Description: Learn how to diagnose and fix 502 Bad Gateway errors when accessing Portainer through a Cloudflare Tunnel, covering connectivity, configuration, WebSocket, and timeout issues.

## Introduction

A 502 Bad Gateway error when accessing Portainer through a Cloudflare Tunnel means cloudflared cannot reach the Portainer backend, or the backend is returning an error that Cloudflare interprets as invalid. These errors can stem from container networking issues, wrong service URLs in the tunnel config, timeout settings, or SSL/TLS configuration mismatches. This guide covers systematic diagnosis and fixes.

## Prerequisites

- Portainer deployed behind a Cloudflare Tunnel
- Access to Docker logs and the Cloudflare Zero Trust dashboard
- cloudflared running as a container or systemd service

## Step 1: Check Cloudflare Tunnel Status

First, verify the tunnel itself is connected:

```bash
# Check cloudflared container logs

docker logs cloudflared --tail=50

# Look for connection state
docker logs cloudflared 2>&1 | grep -E "(registered|disconnected|error|ERR)"

# Healthy output shows 4 registered connections:
# INF Registered tunnel connection connIndex=0
# INF Registered tunnel connection connIndex=1
# INF Registered tunnel connection connIndex=2
# INF Registered tunnel connection connIndex=3

# Problem indicators:
# ERR Failed to dial to origin - Can't reach the backend
# ERR Unable to reach the origin service - Backend not responding
# WRN Retrying connection - Network/auth issues
```

```bash
# Check in Cloudflare Dashboard
# Zero Trust → Networks → Tunnels
# Your tunnel should show: Status = HEALTHY
# If INACTIVE: cloudflared container is not running or not reaching Cloudflare
```

## Step 2: Verify the Service URL in Tunnel Config

The service URL in your tunnel configuration must point to the correct container:

**In config.yml mode:**
```yaml
# /opt/cloudflared/config.yml

ingress:
  - hostname: portainer.example.com
    # WRONG: trying to reach portainer on HTTPS but it's using HTTP
    service: https://portainer:9000    # Port 9000 is HTTP by default

    # CORRECT for Portainer CE on port 9000 (HTTP)
    service: http://portainer:9000

    # CORRECT for Portainer with HTTPS enabled (port 9443)
    service: https://portainer:9443
    originRequest:
      noTLSVerify: true    # Required: Portainer uses self-signed cert
```

**In Cloudflare Dashboard (for token-mode tunnels):**
```text
Public Hostname → Edit service configuration:
  Type: HTTP
  URL: portainer:9000

If using HTTPS:
  Type: HTTPS
  URL: portainer:9443
  No TLS Verify: checked
```

## Step 3: Test Connectivity from Cloudflared Container

```bash
# Confirm cloudflared can reach Portainer
docker exec cloudflared wget -qO- --timeout=10 http://portainer:9000 && echo "OK" || echo "FAILED"

# If FAILED: check they're on the same Docker network
docker inspect cloudflared | jq '.[].NetworkSettings.Networks | keys'
docker inspect portainer | jq '.[].NetworkSettings.Networks | keys'

# They must share a network - connect if not
docker network connect proxy cloudflared
docker network connect proxy portainer

# Retry connectivity test
docker exec cloudflared wget -qO- --timeout=10 http://portainer:9000 && echo "OK"
```

## Step 4: Handle Timeout Issues

Cloudflare Tunnel has default timeouts that may be too short for Portainer operations:

```yaml
# config.yml - Increase timeout settings
ingress:
  - hostname: portainer.example.com
    service: http://portainer:9000
    originRequest:
      connectTimeout: 30s         # Time to establish connection (default: 30s)
      tlsTimeout: 10s             # TLS handshake timeout
      tcpKeepAlive: 30s           # TCP keepalive interval
      keepAliveConnections: 100   # Max keepalive connections
      keepAliveTimeout: 90s       # Idle connection timeout
      httpHostHeader: portainer.example.com
      originServerName: portainer.example.com
```

## Step 5: Fix WebSocket Issues

Portainer's terminal and console use WebSockets. If the connection establishes but console doesn't work:

```yaml
# Ensure WebSockets are enabled in Cloudflare
# Cloudflare Dashboard → your-domain.com → Network → WebSockets → ON

# In tunnel config, WebSocket upgrade is handled automatically by cloudflared
# If issues persist, check timeout:
ingress:
  - hostname: portainer.example.com
    service: http://portainer:9000
    originRequest:
      disableChunkedEncoding: false    # Keep chunked for WebSocket
```

## Step 6: Check Portainer Error Logs

The error may originate from Portainer itself:

```bash
# Check Portainer logs
docker logs portainer --tail=100

# Look for relevant errors
docker logs portainer 2>&1 | grep -i "error\|warn\|fail"

# Verify Portainer is listening
docker exec portainer netstat -tlnp 2>/dev/null | grep -E "9000|9443"
# Expected: 0.0.0.0:9000 (HTTP) or 0.0.0.0:9443 (HTTPS)
```

## Step 7: Use Cloudflare Tunnel Logs for Diagnosis

```bash
# Enable verbose cloudflared logging
# In docker-compose.yml, change command to:
command: tunnel --no-autoupdate --loglevel debug run

# Filter for origin connection issues
docker logs cloudflared 2>&1 | grep -i "origin\|ERR\|upstream"

# Common error messages:
# "dial tcp portainer:9000: connect: connection refused"
#   → Portainer not running or not on the proxy network
# "x509: certificate signed by unknown authority"
#   → Add noTLSVerify: true for self-signed Portainer cert
# "context deadline exceeded"
#   → Increase connectTimeout in originRequest settings
```

## Step 8: Systematic Fix Checklist

```bash
#!/bin/bash
# cloudflare-tunnel-debug.sh

echo "=== Cloudflare Tunnel to Portainer Debug ==="

echo "1. Tunnel container running?"
docker ps --filter "name=cloudflared" --format "  {{.Names}}: {{.Status}}"

echo "2. Portainer container running?"
docker ps --filter "name=portainer" --format "  {{.Names}}: {{.Status}}"

echo "3. Shared Docker network?"
CF_NETS=$(docker inspect cloudflared 2>/dev/null | jq -r '.[].NetworkSettings.Networks | keys[]')
PORT_NETS=$(docker inspect portainer 2>/dev/null | jq -r '.[].NetworkSettings.Networks | keys[]')
echo "  Cloudflared networks: $CF_NETS"
echo "  Portainer networks: $PORT_NETS"

echo "4. Connectivity test:"
docker exec cloudflared wget -qO- --timeout=5 http://portainer:9000 > /dev/null 2>&1 && \
  echo "  SUCCESS: cloudflared can reach portainer" || \
  echo "  FAILED: cloudflared cannot reach portainer"
```

## Conclusion

502 errors with Cloudflare Tunnel and Portainer are almost always caused by cloudflared being unable to reach the Portainer container. Verify they share the same Docker network, confirm the service URL scheme and port are correct (HTTP port 9000 vs HTTPS port 9443), and add `noTLSVerify: true` if Portainer uses a self-signed certificate. Increase timeout settings for environments with slow startup times or heavy operations.
