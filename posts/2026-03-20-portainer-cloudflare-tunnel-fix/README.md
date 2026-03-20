# How to Fix Portainer Not Working Behind Cloudflare Tunnel - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Cloudflare, Tunnel, Reverse Proxy, Troubleshooting

Description: Fix Portainer connectivity issues when deployed behind a Cloudflare Tunnel, including WebSocket failures, container console issues, and log streaming problems.

## Introduction

Cloudflare Tunnel (formerly Argo Tunnel) is a popular way to expose self-hosted services without opening ports. However, Portainer has specific requirements for WebSocket support, HTTP header forwarding, and session handling that require specific Cloudflare Tunnel configuration. This guide covers all common issues.

## Common Issues with Portainer Behind Cloudflare Tunnel

1. Container console (terminal) not working
2. Log streaming stopping after a few seconds
3. "WebSocket connection failed" errors
4. Session drops during container operations
5. "Origin not allowed" errors

## Step 1: Enable WebSocket Support in Cloudflare

```bash
# In the Cloudflare Dashboard:

# 1. Log in to Cloudflare Dashboard
# 2. Go to your domain → Network
# 3. Find "WebSockets"
# 4. Toggle it to "On"

# This is required for:
# - Container console/terminal
# - Real-time log streaming
# - Live stats updates
```

## Step 2: Configure the Cloudflare Tunnel for Portainer

```yaml
# ~/.cloudflared/config.yml or cloudflared tunnel config

tunnel: your-tunnel-id
credentials-file: /path/to/credentials.json

ingress:
  # Portainer configuration
  - hostname: portainer.yourdomain.com
    service: https://localhost:9443
    originRequest:
      # Don't verify self-signed cert
      noTLSVerify: true
      # Keep connections alive for WebSocket
      keepAliveConnections: 10
      keepAliveTimeout: 90s
      # Increase timeout for long operations
      connectTimeout: 30s
      # Set proper headers
      httpHostHeader: portainer.yourdomain.com

  # Catch-all
  - service: http_status:404
```

Deploy with:

```bash
cloudflared tunnel run your-tunnel-name
# or as a service:
sudo cloudflared service install
sudo systemctl start cloudflared
```

## Step 3: Fix Using Cloudflare Zero Trust Dashboard

If using the Cloudflare Zero Trust (ZTNA) dashboard instead of config file:

1. Go to **Zero Trust** → **Access** → **Tunnels**
2. Click your tunnel → **Configure** → **Public Hostname**
3. Add a hostname entry:
   - **Subdomain**: portainer
   - **Domain**: yourdomain.com
   - **Service type**: HTTPS
   - **URL**: localhost:9443
4. Under **Additional application settings** → **TLS Settings**:
   - Enable **No TLS Verify** (for self-signed Portainer cert)
5. Under **Additional application settings** → **HTTP Settings**:
   - **HTTP Host Header**: portainer.yourdomain.com

## Step 4: Fix WebSocket Proxy Configuration

```bash
# Test if WebSocket connections are working
# Install wscat for WebSocket testing
npm install -g wscat

# Test WebSocket connection to Portainer
wscat -c wss://portainer.yourdomain.com
# If connection succeeds: WebSocket is working
# If "Error: Unexpected server response: 400": WebSocket upgrade not configured
```

## Step 5: Fix Cloudflare Cache Interference

```bash
# Cloudflare caching can interfere with API responses
# Create a Page Rule or Cache Rule to bypass cache for Portainer:

# In Cloudflare Dashboard → Rules → Cache Rules
# Create rule:
# Match: hostname portainer.yourdomain.com
# Action: Bypass cache
```

Or add the `Cache-Control: no-store` header configuration in the tunnel config.

## Step 6: Fix Origin Validation Errors

Portainer v2.30+ has stricter origin checking. Configure the tunnel to send correct origin:

```yaml
# cloudflared config.yml
ingress:
  - hostname: portainer.yourdomain.com
    service: https://localhost:9443
    originRequest:
      noTLSVerify: true
      # Tell Portainer what origin to expect
      httpHostHeader: portainer.yourdomain.com
```

Or use Portainer's flag to disable origin checking (less secure):

```bash
docker run -d \
  -p 9443:9443 \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --http-disabled \
  --ssl \
  --sslcert /certs/cert.pem \
  --sslkey /certs/key.pem
```

## Step 7: Fix Timeout for Long Operations

Cloudflare tunnels have default timeouts that affect Portainer operations:

```yaml
# In cloudflared config, increase timeouts
ingress:
  - hostname: portainer.yourdomain.com
    service: https://localhost:9443
    originRequest:
      noTLSVerify: true
      # Increase for long stack deployments
      connectTimeout: 60s
      # 0 = no timeout (for streaming/WebSocket)
      tcpKeepAlive: 30s
```

Cloudflare itself has a maximum timeout of 100 seconds for non-WebSocket connections. For operations that take longer, use the Portainer API with polling rather than waiting for a single long request.

## Step 8: Configure Cloudflare Access Policies (Optional)

Add authentication to your Portainer tunnel:

```bash
# In Cloudflare Zero Trust → Access → Applications → Add Application
# Type: Self-hosted
# Application domain: portainer.yourdomain.com
# Configure policy (e.g., only your email can access)

# This adds Cloudflare Access as an extra auth layer
# Users must authenticate through Cloudflare before reaching Portainer login
```

## Step 9: Fix Docker Compose Deployment for Tunnel

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    # Only expose locally - accessed via Cloudflare Tunnel
    expose:
      - "9443"
    # No public port binding needed with Cloudflare Tunnel
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      TUNNEL_TOKEN: "your-cloudflare-tunnel-token"
    depends_on:
      - portainer

volumes:
  portainer_data:
```

## Step 10: Verify the Full Flow

```bash
# Test 1: Basic connectivity
curl https://portainer.yourdomain.com/api/status

# Test 2: Login
curl -X POST https://portainer.yourdomain.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}'

# Test 3: WebSocket (requires wscat)
wscat -c wss://portainer.yourdomain.com

# If Test 3 fails but 1 and 2 succeed:
# WebSockets are not enabled in Cloudflare Network settings
```

## Conclusion

The most critical requirements for Portainer behind a Cloudflare Tunnel are: WebSocket support enabled in Cloudflare Network settings, `noTLSVerify: true` in the tunnel configuration (for self-signed Portainer certs), and the correct `httpHostHeader` to pass the correct hostname. Container terminal and log streaming will not work without WebSocket support.
