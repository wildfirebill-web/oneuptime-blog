# How to Fix "Origin Invalid" Errors in Portainer Behind a Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Reverse Proxy, Troubleshooting, Security, CSRF

Description: Diagnose and fix the "Origin is not trusted" or "Origin invalid" CSRF protection error when accessing Portainer through a reverse proxy.

## Introduction

When you access Portainer through a reverse proxy, you may encounter an error like:

```
{"message":"Access denied: origin not trusted"}
```

or the login form simply refuses to submit. This is Portainer's CSRF protection working as intended — it rejects requests whose `Origin` header doesn't match the server's expected origin. This guide explains why it happens and how to fix it.

## Why This Error Occurs

Portainer validates the `Origin` header on all state-changing requests. When accessed directly on `https://myserver:9443`, Portainer considers `https://myserver:9443` a trusted origin. When accessed via a reverse proxy at `https://portainer.example.com`, the Origin becomes `https://portainer.example.com` — which Portainer hasn't been told to trust.

## Diagnosing the Issue

Check your browser's developer console for the actual error:

```bash
# Check Portainer logs for origin-related errors
docker logs portainer 2>&1 | grep -i origin

# Test the origin header manually
curl -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -H "Origin: https://portainer.example.com" \
  -d '{"username":"admin","password":"password"}' \
  -v 2>&1 | grep -i "origin\|denied\|trusted"
```

## Fix 1: Use --trusted-origins Flag (Recommended)

Add the `--trusted-origins` flag to your Portainer startup command:

```bash
# Docker run
docker run -d \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com

# Multiple origins (comma-separated)
docker run -d \
  --name portainer \
  portainer/portainer-ce:latest \
  --trusted-origins=https://portainer.example.com,https://portainer.internal.example.com
```

### In Docker Compose

```yaml
  portainer:
    image: portainer/portainer-ce:latest
    command:
      - "--trusted-origins=https://portainer.example.com"
```

### In Docker Swarm Stack

```yaml
  portainer:
    image: portainer/portainer-ce:latest
    command:
      - "--trusted-origins=https://portainer.example.com"
    deploy:
      placement:
        constraints:
          - node.role == manager
```

## Fix 2: Ensure X-Forwarded-Proto Is Set Correctly

Portainer also checks that the scheme in the `Origin` matches what it expects. Make sure your reverse proxy sets the correct `X-Forwarded-Proto` header:

### Nginx

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
```

### Traefik

Traefik sets these headers automatically. Verify with:
```bash
docker exec traefik-container wget -q -O- http://portainer:9000/api/system/status
```

### Apache

```apache
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Host "portainer.example.com"
```

## Fix 3: Check for Double-Encoding or Path Prefix Issues

If Portainer is served on a subpath (e.g., `/portainer/`), the origin may mismatch because of the path. Use `--base-url` alongside `--trusted-origins`:

```yaml
    command:
      - "--base-url=/portainer"
      - "--trusted-origins=https://example.com"
```

## Fix 4: Wildcard Trusted Origins (Not Recommended for Production)

For debugging only, you can trust all origins:

```bash
# WARNING: Disables CSRF protection - only use in isolated dev environments
docker run portainer/portainer-ce:latest --trusted-origins='*'
```

## Verifying the Fix

```bash
# Attempt login via curl - should return a JWT token
curl -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -H "Origin: https://portainer.example.com" \
  -d '{"username":"admin","password":"yourpassword"}'

# Expected response:
# {"jwt":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

## Conclusion

The "Origin invalid" error is a security feature protecting Portainer from CSRF attacks, not a bug. The correct fix is to explicitly configure `--trusted-origins` with the URL your users access Portainer through. Always use the exact URL including scheme and hostname, matching what appears in users' browser address bars.
