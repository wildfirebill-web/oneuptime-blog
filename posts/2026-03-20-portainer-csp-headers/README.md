# How to Configure Content-Security-Policy Headers in Portainer (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, CSP, Headers, Nginx

Description: Learn how to configure Content-Security-Policy and other security headers for Portainer deployments to protect against XSS and clickjacking attacks.

## Introduction

Content-Security-Policy (CSP) and other security headers protect web applications against cross-site scripting (XSS), clickjacking, and other injection attacks. When running Portainer behind a reverse proxy like Nginx or Traefik, you can add these headers at the proxy layer to enhance the security of the Portainer management interface.

## Prerequisites

- Portainer running behind a reverse proxy (Nginx or Traefik)
- HTTPS enabled (required for many security headers to be effective)
- Admin access to the proxy configuration

## Security Headers Overview

| Header | Protection |
|--------|-----------|
| `Content-Security-Policy` | Prevents XSS by restricting resource sources |
| `X-Frame-Options` | Prevents clickjacking (embedding in iframes) |
| `X-Content-Type-Options` | Prevents MIME type sniffing |
| `Strict-Transport-Security` | Forces HTTPS connections |
| `Referrer-Policy` | Controls referrer information leakage |
| `Permissions-Policy` | Restricts browser API access |

## Step 1: Configure Headers in Nginx

```nginx
# /etc/nginx/sites-available/portainer.conf

server {
    listen 443 ssl http2;
    server_name portainer.example.com;

    # TLS configuration
    ssl_certificate /etc/letsencrypt/live/portainer.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portainer.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # ===== Security Headers =====

    # Prevent XSS - restrict content sources to same origin + Portainer CDN
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' 'unsafe-inline' 'unsafe-eval';
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https:;
        font-src 'self' data:;
        connect-src 'self' wss://portainer.example.com;
        frame-ancestors 'none';
    " always;

    # Prevent embedding in iframes (clickjacking)
    add_header X-Frame-Options "DENY" always;

    # Prevent MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Force HTTPS for 1 year, include subdomains
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Control referrer information
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Restrict browser API permissions
    add_header Permissions-Policy "
        camera=(),
        microphone=(),
        geolocation=(),
        payment=(),
        usb=()
    " always;

    # Remove server version from responses
    server_tokens off;
    add_header X-Powered-By "" always;

    location / {
        proxy_pass https://localhost:9443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support for Portainer console
        proxy_read_timeout 900;
    }
}

# HTTP to HTTPS redirect

server {
    listen 80;
    server_name portainer.example.com;
    return 301 https://$host$request_uri;
}
```

## Step 2: Portainer-Specific CSP Considerations

Portainer's UI uses some dynamic features that require careful CSP configuration:

```nginx
# Portainer-compatible CSP
add_header Content-Security-Policy "
    default-src 'self';
    script-src 'self' 'unsafe-inline' 'unsafe-eval';  # Angular.js requires these
    style-src 'self' 'unsafe-inline';                   # Inline styles used in UI
    img-src 'self' data: https: http:;                  # Docker Hub images previewed
    font-src 'self' data:;
    connect-src 'self' wss: https:;                     # WebSocket for terminal
    worker-src blob:;                                    # Web workers if used
    frame-src 'none';                                    # No iframes
    frame-ancestors 'none';                              # Can't be embedded
    object-src 'none';
    base-uri 'self';
    form-action 'self';
" always;
```

## Step 3: Configure Headers in Traefik

```yaml
# traefik-dynamic.yml - Security headers middleware

http:
  middlewares:
    portainer-security-headers:
      headers:
        contentTypeNosniff: true
        browserXssFilter: true
        forceSTSHeader: true
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        frameDeny: true
        referrerPolicy: "strict-origin-when-cross-origin"
        customResponseHeaders:
          X-Powered-By: ""
          Server: ""
        contentSecurityPolicy: "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' wss:; frame-ancestors 'none'; object-src 'none'"
        permissionsPolicy: "camera=(), microphone=(), geolocation=(), payment=()"

  routers:
    portainer:
      rule: "Host(`portainer.example.com`)"
      service: portainer
      middlewares:
        - portainer-security-headers
      tls:
        certResolver: letsencrypt

  services:
    portainer:
      loadBalancer:
        servers:
          - url: "https://portainer:9443"
```

## Step 4: Verify Header Configuration

```bash
# Check security headers are being returned
curl -sI https://portainer.example.com | grep -i -E "(Content-Security|X-Frame|X-Content|Strict|Referrer|Permissions)"

# Expected output:
# content-security-policy: default-src 'self'; ...
# x-frame-options: DENY
# x-content-type-options: nosniff
# strict-transport-security: max-age=31536000; includeSubDomains; preload
# referrer-policy: strict-origin-when-cross-origin

# Use Mozilla Observatory for comprehensive header analysis
# https://observatory.mozilla.org/analyze/portainer.example.com
```

## Step 5: Test with Security Scanners

```bash
# Use nikto for web security scanning
docker run --rm sullo/nikto -h https://portainer.example.com

# Use securityheaders.com API
curl "https://securityheaders.com/?q=portainer.example.com&followRedirects=on"

# Use OWASP ZAP
docker run --rm owasp/zap2docker-stable zap-baseline.py \
  -t https://portainer.example.com \
  -r security-report.html
```

## Step 6: Portainer Native HTTPS Headers

If Portainer is directly exposed (not behind a proxy), some headers can be configured via Portainer startup flags:

```bash
docker run -d \
  --name portainer \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-be:latest \
  --ssl \
  --sslcert /certs/cert.pem \
  --sslkey /certs/key.pem \
  --http-disabled  # Disable plain HTTP entirely
```

## Conclusion

Configuring security headers for Portainer requires adding them at the reverse proxy layer since Portainer doesn't natively expose header configuration via its own settings. Set Content-Security-Policy carefully to allow Portainer's JavaScript requirements while blocking injection attacks, use X-Frame-Options to prevent clickjacking, and enforce HTTPS with HSTS. Regularly test your headers with Mozilla Observatory and keep up with CSP best practices as Portainer updates may change required script sources.
