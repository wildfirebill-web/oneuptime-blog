# How to Deploy an API Gateway with Portainer - Deploy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, API Gateway, Kong, Nginx, Microservice, Docker Compose, Rate Limiting

Description: Learn how to deploy Kong as an API gateway via Portainer for rate limiting, authentication, and request routing to backend microservices.

---

An API gateway centralizes cross-cutting concerns: authentication, rate limiting, logging, and routing. Kong is the most popular open-source API gateway. Portainer manages the multi-container deployment (Kong, PostgreSQL, and a dashboard).

## Compose Stack

```yaml
version: "3.8"

services:
  kong-db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: kong
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kongpass        # Change this
    volumes:
      - kong_db:/var/lib/postgresql/data

  kong-migration:
    image: kong:3.6
    restart: on-failure
    depends_on:
      - kong-db
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PG_DATABASE: kong
    # Run migrations once and exit
    command: kong migrations bootstrap

  kong:
    image: kong:3.6
    restart: unless-stopped
    depends_on:
      - kong-db
    ports:
      - "8000:8000"    # HTTP proxy
      - "8443:8443"    # HTTPS proxy
      - "8001:8001"    # Admin API
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PG_DATABASE: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001

  konga:
    image: pantsel/konga:latest
    restart: unless-stopped
    ports:
      - "1337:1337"    # Konga dashboard
    environment:
      NODE_ENV: production

volumes:
  kong_db:
```

## Configuring Routes via Kong Admin API

After deployment, configure Kong to route to your services:

```bash
# Create a service pointing to your backend

curl -X POST http://localhost:8001/services \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user-service",
    "url": "http://user-service:3001"
  }'

# Create a route for the service
curl -X POST http://localhost:8001/services/user-service/routes \
  -H "Content-Type: application/json" \
  -d '{
    "paths": ["/api/users"],
    "strip_path": true
  }'
```

## Adding Rate Limiting Plugin

```bash
# Add rate limiting to the user-service (10 requests per minute)
curl -X POST http://localhost:8001/services/user-service/plugins \
  -H "Content-Type: application/json" \
  -d '{
    "name": "rate-limiting",
    "config": {
      "minute": 10,
      "policy": "local"
    }
  }'
```

## Monitoring

Use OneUptime to monitor `http://<host>:8001/status`. Kong returns an extensive health status JSON. Alert on any degraded state to catch database connectivity issues before they affect API traffic.
