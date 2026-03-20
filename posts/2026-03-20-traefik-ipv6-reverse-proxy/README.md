# How to Configure Traefik as an IPv6 Reverse Proxy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Traefik, Reverse Proxy, Kubernetes, Docker, Dual-Stack

Description: Configure Traefik as an IPv6-capable reverse proxy with dual-stack entry points, IPv6 middleware for client IP, and routing to IPv6 backend services.

## Introduction

Traefik is a cloud-native reverse proxy popular in Docker and Kubernetes environments. IPv6 support requires configuring entry points with IPv6 addresses, enabling proper `X-Forwarded-For` processing for IPv6 clients, and ensuring service discovery returns IPv6 endpoints.

## Static Configuration (IPv6 Entry Points)

```yaml
# /etc/traefik/traefik.yml

entryPoints:
  web:
    address: "[::]:80"        # Listen on all IPv6 (and IPv4 via dual-stack)
  websecure:
    address: "[::]:443"
    http:
      tls:
        certResolver: letsencrypt

  # IPv6-only entry point
  web-ipv6:
    address: "[2001:db8::1]:80"

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: /var/traefik/acme.json
      httpChallenge:
        entryPoint: web

# Real IP configuration
entryPoints:
  websecure:
    address: "[::]:443"
    forwardedHeaders:
      trustedIPs:
        - "10.0.0.0/8"
        - "fd00::/8"
        - "2001:db8:lb::/48"
```

## Docker Provider Configuration

```yaml
# docker-compose.yml — Traefik with IPv6

services:
  traefik:
    image: traefik:v3.0
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=[::]:80
      - --entrypoints.websecure.address=[::]:443
    ports:
      - "80:80"
      - "[::]:80:80"
      - "443:443"
      - "[::]:443:443"
    networks:
      - ipv6-net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  app:
    image: my-app:latest
    networks:
      - ipv6-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      - "traefik.http.services.app.loadbalancer.server.port=8080"

networks:
  ipv6-net:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: "fd00:traefik::/64"
```

## Kubernetes IngressRoute with IPv6

```yaml
# traefik IngressRoute for Kubernetes
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: app-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: app-service
          port: 8080
  tls:
    certResolver: letsencrypt

---
# Dual-stack service for Traefik to discover
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: my-app
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv6
    - IPv4
  ports:
    - port: 8080
      targetPort: 8080
```

## Middleware for IPv6 Client IP

```yaml
# Traefik middleware to extract real client IPv6 IP
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: real-ip-middleware
spec:
  # Traefik has built-in real IP handling via entryPoint.forwardedHeaders
  # This middleware adds additional headers
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
```

```yaml
# Dynamic configuration for IP allowlist (IPv6)
http:
  middlewares:
    internal-ipv6-only:
      ipAllowList:
        sourceRange:
          - "fd00::/8"       # ULA internal
          - "2001:db8::/32"  # Organization
          - "10.0.0.0/8"     # IPv4 internal (for dual-stack)
```

## Verify IPv6 Entry Points

```bash
# Check Traefik API for entry point configuration
curl -6 "http://[::1]:8080/api/entrypoints" | python3 -m json.tool

# Test IPv6 access
curl -6 -v https://app.example.com/health

# Check Traefik logs for IPv6 connections
docker logs traefik 2>&1 | grep -E '[0-9a-fA-F:]{3,39}'
```

## Conclusion

Traefik IPv6 configuration centers on the `address: "[::]:port"` format for entry points, which listens on all IPv6 (and IPv4 via dual-stack) interfaces. Configure `forwardedHeaders.trustedIPs` with both IPv4 and IPv6 CIDR ranges for accurate client IP extraction. In Kubernetes, Traefik discovers service endpoints — ensure services use `ipFamilyPolicy: PreferDualStack` to expose IPv6 endpoints for Traefik to route to. The Docker provider automatically uses container IPv6 addresses when the network is IPv6-enabled.
