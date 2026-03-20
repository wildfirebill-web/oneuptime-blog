# How to Configure Traefik for IPv6 Load Balancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Traefik, IPv6, Load Balancing, Reverse Proxy, Docker, Kubernetes

Description: A guide to configuring Traefik reverse proxy and load balancer with IPv6 support, including entrypoints, middleware, and backend routing over IPv6.

Traefik is a modern HTTP reverse proxy and load balancer that supports IPv6 natively. IPv6 is configured at the entrypoint level (where Traefik listens) and at the service level (where Traefik connects to backends).

## Static Configuration: IPv6 Entrypoints

```yaml
# traefik.yml (static configuration)

entryPoints:
  # Listen on all interfaces (IPv4 and IPv6)
  web:
    address: ":80"           # :: (all interfaces) on port 80

  websecure:
    address: ":443"          # :: on port 443

  # Listen on specific IPv6 address only
  ipv6-only:
    address: "[::]:80"       # Explicit IPv6 all-interfaces

  # Listen on specific IPv6 address
  ipv6-specific:
    address: "[2001:db8::proxy]:80"
```

## Docker Compose with IPv6 Traefik

```yaml
# docker-compose.yml

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - traefik-net

  webapp:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.webapp.rule=Host(`example.com`)"
      - "traefik.http.routers.webapp.entrypoints=websecure"
      - "traefik.http.services.webapp.loadbalancer.server.port=80"
    networks:
      - traefik-net

networks:
  traefik-net:
    driver: bridge
    enable_ipv6: true
    ipam:
      config:
        - subnet: "fd00:traefik::/64"
```

## IPv6 Backend Services

```yaml
# Dynamic configuration: traefik/dynamic.yml

http:
  services:
    ipv6-backend:
      loadBalancer:
        servers:
          # IPv6 backend servers (use brackets for IPv6 in URLs)
          - url: "http://[2001:db8::server1]:8080"
          - url: "http://[2001:db8::server2]:8080"

        healthCheck:
          path: /health
          interval: 10s
          timeout: 5s

  routers:
    ipv6-route:
      rule: "Host(`api.example.com`)"
      service: ipv6-backend
      entryPoints:
        - websecure
      tls: {}
```

## Kubernetes IngressRoute with IPv6 Backends

```yaml
# Traefik IngressRoute for Kubernetes

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: ipv6-app
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: my-service
          port: 80
  tls:
    certResolver: letsencrypt
---
# Kubernetes Service with IPv6 ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  # For dual-stack Kubernetes clusters:
  ipFamilies:
    - IPv6
    - IPv4
  ipFamilyPolicy: PreferDualStack
```

## IPv6 Real IP Forwarding

Configure Traefik to pass the real IPv6 client IP to backends:

```yaml
# traefik.yml

entryPoints:
  web:
    address: ":80"
    forwardedHeaders:
      trustedIPs:
        - "2001:db8:proxy::/64"    # Trusted proxy IPv6 range
        - "::1/128"                 # Localhost

http:
  middlewares:
    real-ip:
      headers:
        customRequestHeaders:
          X-Real-IP: ""             # Forward original IP
```

## Verifying IPv6 Traefik

```bash
# Start Traefik
docker compose up -d

# Verify Traefik listens on IPv6
ss -6 -tlnp | grep traefik

# Test IPv6 access
curl -6 http://[::1]/health
curl -6 https://example.com/

# Check Traefik dashboard
curl http://localhost:8080/api/rawdata | python3 -m json.tool | grep -A 5 "entryPoints"
```

## Troubleshooting IPv6 in Traefik

```bash
# Check if Docker network has IPv6
docker network inspect traefik-net | grep -A 5 "IPv6"

# Enable IPv6 in Docker daemon
# /etc/docker/daemon.json:
# {"ipv6": true, "fixed-cidr-v6": "fd00:docker::/80"}

# Verify backend container has IPv6 address
docker inspect webapp | grep "GlobalIPv6Address"
```

Traefik's automatic service discovery combined with native IPv6 support makes it straightforward to deploy IPv6-capable load balancing for containerized applications with minimal configuration.
