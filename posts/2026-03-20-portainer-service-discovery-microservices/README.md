# How to Configure Service Discovery for Microservices in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Service Discovery, Microservices, Docker, Consul, Traefik, DNS

Description: Learn how to configure service discovery for microservices in Portainer using Docker's built-in DNS, Consul, or Traefik for dynamic service registration.

---

Service discovery allows microservices to find each other without hardcoded IP addresses. Docker provides basic DNS-based service discovery for free; for advanced scenarios with health-check-based routing, tools like Consul or Traefik's service discovery add more capability.

## Level 1: Docker DNS (Built-in, No Extra Tools)

Docker's embedded DNS resolver is sufficient for most cases:

```yaml
version: "3.8"
services:
  order-service:
    image: order-service:latest
    networks:
      - app_network

  inventory-service:
    image: inventory-service:latest
    networks:
      - app_network

networks:
  app_network:
```

```javascript
// order-service/src/inventory.js
// Discover inventory-service by its service name
const inventoryUrl = 'http://inventory-service:3002';
```

## Level 2: Traefik as Service Discovery + Load Balancer

Traefik auto-discovers services via Docker labels and routes traffic dynamically:

```yaml
version: "3.8"
services:
  traefik:
    image: traefik:v3.0
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedByDefault=false
    ports:
      - "80:80"
      - "8081:8080"    # Traefik dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  user-service:
    image: user-service:latest
    labels:
      - "traefik.enable=true"
      # Route /api/users to this service
      - "traefik.http.routers.users.rule=PathPrefix(`/api/users`)"
      - "traefik.http.services.users.loadbalancer.server.port=3001"

  product-service:
    image: product-service:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.products.rule=PathPrefix(`/api/products`)"
      - "traefik.http.services.products.loadbalancer.server.port=3002"
```

## Level 3: Consul for Dynamic Service Registry

For complex environments with health-check-based failover:

```yaml
  consul:
    image: consul:1.16
    restart: unless-stopped
    ports:
      - "8500:8500"    # Consul UI
    command: agent -dev -ui -client=0.0.0.0
```

Register services with Consul on startup:

```bash
# Register a service with health check

curl -X PUT http://consul:8500/v1/agent/service/register \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "user-service",
    "Port": 3001,
    "Check": {
      "HTTP": "http://localhost:3001/health",
      "Interval": "10s"
    }
  }'
```

```javascript
// Discover a service via Consul DNS
// Consul provides DNS at: <service-name>.service.consul
const inventoryUrl = 'http://inventory-service.service.consul:3002';
```

## Monitoring Service Discovery

Use OneUptime to monitor each service's health endpoint. If service discovery breaks (e.g., a service becomes unhealthy but Consul doesn't deregister it), you'll catch it through monitoring before dependent services start failing.
