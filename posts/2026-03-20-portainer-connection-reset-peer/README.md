# How to Fix 'Connection Reset by Peer' Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Networking, Reverse Proxy

Description: Diagnose and fix 'Connection Reset by Peer' errors in Portainer, which commonly occur when using reverse proxies, WebSocket connections, or TLS misconfiguration.

## Introduction

"Connection Reset by Peer" (TCP RST) errors in Portainer typically indicate that a connection was established but then forcibly closed by the remote side. This is different from "Connection Refused" (which means no connection was established at all). The most common causes are reverse proxy timeouts, WebSocket upgrade failures, and TLS handshake issues.

## Common Causes

1. Reverse proxy timeout closing idle WebSocket connections
2. Missing WebSocket upgrade headers in proxy configuration
3. TLS certificate mismatch or SNI issues
4. Portainer session timeout
5. Network equipment (load balancers, firewalls) closing long-lived connections

## Step 1: Identify When the Error Occurs

```bash
# Check Portainer logs during the error

docker logs portainer -f --tail 50

# Check for WebSocket-related errors
docker logs portainer 2>&1 | grep -i "websocket\|reset\|pipe\|EOF"
```

## Scenario 1: Error Occurs in Browser Console

If you see "ERR_CONNECTION_RESET" in Chrome or Firefox:

```bash
# Open browser developer tools → Network tab
# Look for failed WebSocket connections (protocol: ws:// or wss://)
# These are used for:
# - Container console/terminal
# - Log streaming
# - Real-time stats
```

This almost always indicates a reverse proxy WebSocket configuration issue.

## Scenario 2: Nginx Reverse Proxy - Fix WebSocket Support

```nginx
server {
    listen 443 ssl;
    server_name portainer.yourdomain.com;

    # Required for WebSocket connections in Portainer
    location / {
        proxy_pass https://localhost:9443;

        proxy_http_version 1.1;

        # WebSocket upgrade headers - CRITICAL
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Increase timeouts for long-running operations
        proxy_read_timeout 900s;
        proxy_connect_timeout 60s;
        proxy_send_timeout 900s;

        # Disable buffering for real-time log streaming
        proxy_buffering off;
        proxy_cache off;

        # Allow large request bodies for image uploads
        client_max_body_size 1000m;
    }
}
```

## Scenario 3: Apache Reverse Proxy - Fix WebSocket Support

```apache
<VirtualHost *:443>
    ServerName portainer.yourdomain.com

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/portainer.crt
    SSLCertificateKeyFile /etc/ssl/private/portainer.key

    # Enable required Apache modules
    # a2enmod proxy proxy_http proxy_wstunnel rewrite ssl

    # WebSocket proxying
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteRule /(.*) wss://localhost:9443/$1 [P,L]

    ProxyPass / https://localhost:9443/
    ProxyPassReverse / https://localhost:9443/

    # Headers
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPreserveHost On

    # Timeout settings
    ProxyTimeout 900
    Timeout 900
</VirtualHost>
```

## Scenario 4: Traefik - Fix Connection Reset

```yaml
# traefik.yml dynamic configuration
http:
  routers:
    portainer:
      rule: "Host(`portainer.yourdomain.com`)"
      service: portainer
      tls:
        certResolver: letsencrypt

  services:
    portainer:
      loadBalancer:
        servers:
          - url: "https://portainer:9443"
        # Disable sticky sessions if causing issues
        sticky: null

  middlewares:
    portainer-headers:
      headers:
        # Required for WebSocket
        customRequestHeaders:
          X-Forwarded-Proto: "https"
```

## Scenario 5: Cloudflare - Connection Reset by Peer

Cloudflare's default WebSocket support settings can cause issues:

```bash
# In Cloudflare dashboard:
# 1. Network → WebSockets → Enable
# 2. SSL/TLS → Edge Certificates → Minimum TLS Version → TLS 1.2
# 3. Speed → Optimization → Disable Rocket Loader for Portainer domain

# For the Cloudflare tunnel, ensure WebSocket support:
# Tunnels → Configuration → Add Public Hostname
# Service: https://portainer:9443
# Additional settings → HTTP Host Header: portainer.yourdomain.com
```

## Scenario 6: AWS ALB / Load Balancer

For AWS Application Load Balancers:

1. Ensure the target group protocol is HTTPS
2. Set idle timeout to at least 600 seconds (default is 60)
3. Enable sticky sessions if needed:

```bash
# AWS CLI: update ALB target group idle timeout
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=idle_timeout.timeout_seconds,Value=600
```

## Scenario 7: Check for Network MTU Issues

```bash
# MTU mismatches can cause TCP RST
# Check Docker network MTU
docker network inspect bridge | grep '"com.docker.network.driver.mtu"'

# Check host interface MTU
ip link show eth0

# If Docker containers have different MTU than host
cat > /etc/docker/daemon.json << 'EOF'
{
  "mtu": 1450
}
EOF

sudo systemctl restart docker
```

## Conclusion

"Connection Reset by Peer" in Portainer is almost always a proxy configuration issue. The most critical fix is ensuring your reverse proxy has proper WebSocket upgrade headers (`Upgrade` and `Connection`) and sufficient timeouts for long-running operations like log streaming and container terminals. Apply the appropriate fix for your proxy (Nginx, Apache, Traefik, or Cloudflare) and the errors should resolve immediately.
