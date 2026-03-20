# How to Troubleshoot 502 Bad Gateway Errors in Nginx with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Nginx, 502 Error, Troubleshooting, Docker, Reverse Proxy

Description: Learn how to diagnose and fix 502 Bad Gateway errors when using Nginx as a reverse proxy in front of Portainer, covering common causes and solutions.

## Introduction

A 502 Bad Gateway error from Nginx means the proxy successfully connected to Nginx, but Nginx could not connect to the upstream backend (Portainer). This guide covers the most common causes and how to resolve them.

## Step 1: Check if Portainer is Running

```bash
docker ps | grep portainer
docker inspect portainer | grep '"Status"'
```

If Portainer is stopped or restarting:

```bash
docker logs portainer --tail=50
docker start portainer
```

## Step 2: Verify Portainer is Listening

Check the port Portainer is bound to:

```bash
docker port portainer
# or
ss -tlnp | grep 9000
```

## Step 3: Test Direct Connectivity to Portainer

From the host or another container:

```bash
curl -v http://localhost:9000
# or if using Docker network
curl -v http://portainer:9000
```

If this fails, the problem is with Portainer, not Nginx.

## Step 4: Check Nginx Configuration

Review the proxy_pass directive:

```nginx
location / {
    proxy_pass http://portainer:9000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Common issues:
- Wrong port (Portainer CE: 9000, Portainer BE: 9443 for HTTPS)
- Wrong hostname (must match Docker network service name)
- Missing trailing slash inconsistency

## Step 5: Check Network Connectivity

Ensure Nginx and Portainer are on the same Docker network:

```bash
docker network inspect bridge
docker inspect portainer | grep Networks -A10
docker inspect nginx | grep Networks -A10
```

If they're on different networks:

```bash
docker network connect proxy portainer
```

## Step 6: Review Nginx Error Logs

```bash
docker logs nginx --tail=100
# or
docker exec nginx cat /var/log/nginx/error.log | tail -50
```

Look for messages like:
- `connect() failed (111: Connection refused)`
- `no live upstreams while connecting to upstream`
- `upstream timed out`

## Step 7: Check WebSocket Support

Portainer uses WebSockets. Add WebSocket headers to Nginx:

```nginx
location / {
    proxy_pass http://portainer:9000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;
}
```

## Step 8: Restart Both Services

```bash
docker restart portainer nginx
```

## Conclusion

502 errors between Nginx and Portainer usually stem from network connectivity issues, wrong ports, or missing WebSocket headers. Systematically checking each layer — from container health to network membership to Nginx configuration — quickly identifies the root cause.
