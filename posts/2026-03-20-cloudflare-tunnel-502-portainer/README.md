# How to Fix 502 Bad Gateway Errors with Cloudflare Tunnel and Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Cloudflare, Tunnel, 502 Bad Gateway, Troubleshooting

Description: Learn how to diagnose and fix 502 Bad Gateway errors when accessing Portainer through a Cloudflare Tunnel, covering network issues, timeouts, and misconfigured service URLs.

## Understanding 502 with Cloudflare Tunnel

A 502 from Cloudflare Tunnel means the cloudflared connector reached your server but couldn't forward the request to Portainer. This is different from a regular Nginx 502 — the request successfully traverses the tunnel but fails at the last hop.

```
Browser → Cloudflare → cloudflared connector → Portainer
                                              ↑ This hop fails → 502
```

## Common Causes

| Cause | Symptom |
|-------|---------|
| Portainer not running | cloudflared logs: "connect: connection refused" |
| Wrong service URL in tunnel config | 502 immediately on all requests |
| cloudflared not on same network as Portainer | "no route to host" |
| Portainer HTTP vs HTTPS mismatch | SSL handshake errors |

## Step 1: Check Portainer Is Running

```bash
docker ps | grep portainer
# Must show "Up" status

# Test Portainer is responding locally
curl -I http://localhost:9000
# Expected: HTTP/1.1 200 OK
```

## Step 2: Verify cloudflared Can Reach Portainer

```bash
# If both are containers, check they share a network
docker exec cloudflared curl -s http://portainer:9000
# Or if cloudflared runs as a host service:
curl -s http://localhost:9000
```

## Step 3: Check Tunnel Service URL Configuration

In Cloudflare Zero Trust → **Networks → Tunnels → Configure**:

```
Public Hostname:  portainer.yourdomain.com
Service:          http://localhost:9000     ← if cloudflared runs on host
                  http://portainer:9000     ← if cloudflared runs in Docker
```

Common mistakes:
- Using `https://localhost:9000` when Portainer uses HTTP
- Using `portainer:9000` but cloudflared is on a different Docker network
- Wrong port (9443 vs 9000)

## Step 4: Fix Docker Network Issue

If cloudflared is a container and uses container name routing:

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    networks:
      - default    # Must match Portainer's network

  portainer:
    image: portainer/portainer-ce:latest
    networks:
      - default    # Same network

# Tunnel config service URL: http://portainer:9000
```

## Step 5: Handle HTTP vs HTTPS

If Portainer runs with HTTPS (port 9443):

In Cloudflare tunnel public hostname:
```
Service: https://localhost:9443
```

Also enable in Cloudflare tunnel settings:
- **No TLS Verify**: ON (for self-signed Portainer certs)

Or via cloudflared config:

```yaml
# config.yml
ingress:
  - hostname: portainer.yourdomain.com
    service: https://localhost:9443
    originRequest:
      noTLSVerify: true
```

## Step 6: Check cloudflared Logs

```bash
# If running as Docker container
docker logs cloudflared 2>&1 | tail -30

# Look for connection errors:
# level=error msg="error dialing service URL"
# level=error msg="connect: connection refused"
```

## Step 7: Increase Cloudflare Tunnel Timeouts

For Portainer operations that take longer (large image pulls):

```yaml
# cloudflared config.yml
ingress:
  - hostname: portainer.yourdomain.com
    service: http://localhost:9000
    originRequest:
      connectTimeout: 30s
      tlsTimeout: 30s
      tcpKeepAlive: 30s
      keepAliveTimeout: 90s
      keepAliveConnections: 10
```

## Quick Diagnostic Script

```bash
#!/bin/bash
echo "=== Cloudflare Tunnel + Portainer Diagnostics ==="

echo -e "\n1. Portainer container status:"
docker ps --filter name=portainer --format "{{.Names}}: {{.Status}}"

echo -e "\n2. Portainer local response:"
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://localhost:9000

echo -e "\n3. cloudflared container status:"
docker ps --filter name=cloudflared --format "{{.Names}}: {{.Status}}"

echo -e "\n4. cloudflared logs (last 10 lines):"
docker logs cloudflared 2>&1 | tail -10

echo -e "\n5. Network connectivity test:"
docker exec cloudflared curl -s -o /dev/null -w "From cloudflared to portainer: %{http_code}\n" http://portainer:9000 2>/dev/null || echo "Cannot reach portainer by name"
```

## Conclusion

502 errors with Cloudflare Tunnel are almost always a misconfigured service URL, a Docker network isolation issue, or a Portainer HTTP/HTTPS mismatch. The key diagnostic is testing connectivity from within the cloudflared container to the Portainer container — if that succeeds, the problem is in the tunnel's service configuration; if it fails, it's a networking issue.
