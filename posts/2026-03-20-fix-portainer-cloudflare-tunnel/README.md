# How to Fix Portainer Not Working Behind Cloudflare Tunnel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Cloudflare Tunnel, Reverse Proxy, WebSocket, Networking

Description: Learn how to fix Portainer UI and console issues when accessed through a Cloudflare Tunnel, including WebSocket configuration and HTTP/2 compatibility settings.

---

Cloudflare Tunnel provides a secure way to expose Portainer without opening inbound firewall ports. However, the tunnel's HTTP proxying can break WebSocket connections (needed for the container console) and cause origin validation errors.

## Step 1: Configure Cloudflare Tunnel for WebSocket

In the Cloudflare Zero Trust dashboard:

1. Go to **Access > Tunnels > [your tunnel] > Public Hostnames**.
2. Edit the Portainer hostname.
3. Under **Additional application settings**, enable **WebSocket support**.

## Step 2: Set the Tunnel Origin to HTTP, Not HTTPS

Portainer inside Docker is typically plain HTTP (port 9000). Set the tunnel origin accordingly:

```text
# In Cloudflare Tunnel configuration (config.yml)

ingress:
  - hostname: portainer.example.com
    service: http://portainer:9000    # Use http, not https
    originRequest:
      noTLSVerify: false
      httpHostHeader: portainer.example.com
```

## Step 3: Fix "Origin Invalid" Errors

Cloudflare forwards requests with the original `Host` header but Portainer's CSRF protection checks the `Origin` header. Configure Portainer to trust Cloudflare:

```bash
# Add the --http-enabled flag to ensure Portainer accepts HTTP connections
# from the trusted Cloudflare tunnel
docker run -d ... portainer/portainer-ce:latest \
  --http-enabled \
  --tunnel-addr 0.0.0.0
```

## Step 4: Set Cloudflare SSL/TLS Mode

Ensure Cloudflare's SSL/TLS mode is appropriate:

1. Go to **Cloudflare Dashboard > SSL/TLS**.
2. Set mode to **Full** (not Full Strict, since Portainer uses self-signed cert internally).
3. This ensures HTTPS from browser to Cloudflare but allows HTTP within the tunnel.

## Step 5: Disable Cloudflare HTTP/2 for Portainer Hostname

Portainer's WebSocket implementation may conflict with HTTP/2 multiplexing:

1. In Cloudflare Dashboard go to **Speed > Optimization > Protocol Optimization**.
2. For the Portainer hostname, consider disabling HTTP/2 if WebSocket issues persist.

## Step 6: Fix Container Console Behind Cloudflare

The container console uses a long-lived WebSocket. Cloudflare has a default 100-second timeout on WebSocket connections. Configure idle timeout:

1. In **Cloudflare Dashboard > Network > WebSockets** ensure WebSockets are enabled.
2. Consider using Cloudflare Access policies to restrict who can reach Portainer while keeping the tunnel functional.

## Testing the Fix

```bash
# Test WebSocket connection through the tunnel
wscat -c wss://portainer.example.com/api/websocket

# Should connect without TLS handshake errors
```
