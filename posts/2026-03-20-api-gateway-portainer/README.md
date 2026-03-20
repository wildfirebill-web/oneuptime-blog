# How to Deploy an API Gateway with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, API Gateway, Kong, Traefik, Microservices, Security

Description: Deploy Kong or Traefik as a production-ready API gateway with rate limiting, authentication plugins, and analytics using Portainer.

## Introduction

An API gateway manages all incoming API traffic, enforcing security policies, rate limits, and routing rules in a single place. This guide covers deploying Kong Gateway - the most feature-rich open-source API gateway - alongside Portainer for management and visibility.

## Step 1: Deploy Kong Gateway with PostgreSQL

```yaml
# docker-compose.yml - Kong API Gateway

version: "3.8"

networks:
  kong_net:
    driver: bridge

volumes:
  kong_db:

services:
  # Kong database
  kong_db:
    image: postgres:15-alpine
    container_name: kong_db
    restart: unless-stopped
    environment:
      - POSTGRES_DB=kong
      - POSTGRES_USER=kong
      - POSTGRES_PASSWORD=kong_password
    volumes:
      - kong_db:/var/lib/postgresql/data
    networks:
      - kong_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kong"]
      interval: 10s
      retries: 5

  # Kong database migrations
  kong_migrations:
    image: kong:3.6
    command: kong migrations bootstrap
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong_db
      - KONG_PG_USER=kong
      - KONG_PG_PASSWORD=kong_password
      - KONG_PG_DATABASE=kong
    networks:
      - kong_net
    depends_on:
      kong_db:
        condition: service_healthy
    restart: "no"

  # Kong gateway
  kong:
    image: kong:3.6
    container_name: kong
    restart: unless-stopped
    depends_on:
      kong_db:
        condition: service_healthy
    ports:
      - "8000:8000"   # HTTP proxy
      - "8443:8443"   # HTTPS proxy
      - "8001:8001"   # Admin API (restrict in production)
      - "8444:8444"   # Admin API HTTPS
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong_db
      - KONG_PG_USER=kong
      - KONG_PG_PASSWORD=kong_password
      - KONG_PG_DATABASE=kong
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
      - KONG_ADMIN_GUI_URL=http://localhost:8002
    networks:
      - kong_net

  # Konga - Kong Admin UI
  konga:
    image: pantsel/konga:latest
    container_name: konga
    restart: unless-stopped
    ports:
      - "1337:1337"
    environment:
      - NODE_ENV=production
      - DB_ADAPTER=postgres
      - DB_HOST=kong_db
      - DB_PORT=5432
      - DB_USER=kong
      - DB_PASSWORD=kong_password
      - DB_DATABASE=konga
    networks:
      - kong_net
    depends_on:
      - kong_db
```

## Step 2: Configure Kong Routes and Services

```bash
# Create a service (upstream API)
curl -X POST http://localhost:8001/services \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user-service",
    "url": "http://user_service:8002"
  }'

# Create a route for the service
curl -X POST http://localhost:8001/services/user-service/routes \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user-route",
    "paths": ["/api/v1/users"],
    "strip_path": false,
    "protocols": ["http", "https"]
  }'

# Add another service
curl -X POST http://localhost:8001/services \
  -d "name=order-service" \
  -d "url=http://order_service:8003"

# Route for order service
curl -X POST http://localhost:8001/services/order-service/routes \
  -d "paths[]=/api/v1/orders" \
  -d "strip_path=false"
```

## Step 3: Add Rate Limiting Plugin

```bash
# Global rate limiting (applies to all routes)
curl -X POST http://localhost:8001/plugins \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rate-limiting",
    "config": {
      "minute": 100,
      "hour": 1000,
      "policy": "redis",
      "redis_host": "redis",
      "redis_port": 6379
    }
  }'

# Rate limiting on specific route
curl -X POST http://localhost:8001/routes/user-route/plugins \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rate-limiting",
    "config": {
      "minute": 20,
      "limit_by": "consumer",
      "policy": "local"
    }
  }'
```

## Step 4: Add JWT Authentication

```bash
# Enable JWT plugin on route
curl -X POST http://localhost:8001/routes/user-route/plugins \
  -d "name=jwt"

# Create a consumer
curl -X POST http://localhost:8001/consumers \
  -d "username=my-application"

# Create JWT credentials for consumer
curl -X POST http://localhost:8001/consumers/my-application/jwt \
  -d "algorithm=HS256" \
  -d "secret=my-secret-key"

# Get the key_id
curl http://localhost:8001/consumers/my-application/jwt | jq '.data[0].key'
```

## Step 5: Add Request/Response Transformation

```bash
# Add headers to all requests
curl -X POST http://localhost:8001/plugins \
  -d "name=request-transformer" \
  -d "config.add.headers=X-Gateway:Kong" \
  -d "config.add.headers=X-Request-Time:$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# Remove sensitive response headers
curl -X POST http://localhost:8001/plugins \
  -d "name=response-transformer" \
  -d "config.remove.headers=Server" \
  -d "config.remove.headers=X-Powered-By"
```

## Step 6: Add Prometheus Monitoring

```bash
# Enable Prometheus metrics export
curl -X POST http://localhost:8001/plugins \
  -d "name=prometheus"

# Metrics available at:
# GET http://localhost:8001/metrics
```

```yaml
# Add to prometheus.yml scrape configs
scrape_configs:
  - job_name: 'kong'
    static_configs:
      - targets: ['kong:8001']
    metrics_path: '/metrics'
```

## Monitoring in Portainer

View Kong logs in Portainer:
1. Navigate to **Containers** > **kong** > **Logs**
2. Filter for error entries:

```bash
# Filter Kong error logs
docker logs kong 2>&1 | grep -i error

# Watch access logs
docker logs -f kong 2>&1 | grep "HTTP/1"
```

## Conclusion

Kong Gateway provides enterprise-grade API management for your microservices. With Portainer managing the containers, you get both powerful API governance (rate limiting, authentication, transformation) and operational visibility. The Konga UI simplifies Kong administration, while Portainer handles container lifecycle management, log viewing, and health monitoring. For production deployments, consider Kong's declarative configuration (deck) to version-control your gateway configuration.
