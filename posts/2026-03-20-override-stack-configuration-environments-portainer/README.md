# How to Override Stack Configuration for Different Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Compose, Environment Variables, Stacks, DevOps, Multi-Environment

Description: Use Docker Compose override files and Portainer's environment variable editor to maintain a single stack definition that deploys differently across development, staging, and production environments.

---

Managing separate stack files for development, staging, and production leads to configuration drift. Docker Compose's override mechanism and Portainer's environment variable injection let you maintain a single canonical stack file with environment-specific values applied at deploy time.

## Override Strategy Options

| Approach | Mechanism |
|---|---|
| Environment variables | Portainer's env editor per stack |
| Compose override files | `docker-compose.override.yml` |
| `.env` files | Portainer environment variable files |
| Portainer Stack per environment | Separate stack instances with different vars |

## Step 1: Parameterize the Base Stack

Use variable substitution in your stack YAML so values can vary per environment:

```yaml
# base-stack.yml — parameterized for any environment
version: "3.8"
services:
  api:
    image: my-registry/api:${IMAGE_TAG:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - REPLICAS=${API_REPLICAS:-2}
    deploy:
      replicas: ${API_REPLICAS:-2}
      resources:
        limits:
          memory: ${API_MEMORY_LIMIT:-512m}
    ports:
      - "${API_PORT:-8080}:8080"

  redis:
    image: redis:${REDIS_VERSION:-7-alpine}
    command: redis-server --maxmemory ${REDIS_MAX_MEMORY:-256mb}
    restart: unless-stopped
```

## Step 2: Set Environment Variables per Stack in Portainer

When deploying the stack in Portainer (**Stacks > Add Stack**), scroll to the **Environment variables** section:

**Development values:**
```
IMAGE_TAG=dev-latest
DATABASE_URL=postgresql://dev-db:5432/myapp_dev
LOG_LEVEL=debug
API_REPLICAS=1
API_MEMORY_LIMIT=256m
API_PORT=8080
REDIS_VERSION=7-alpine
REDIS_MAX_MEMORY=128mb
```

**Production values:**
```
IMAGE_TAG=v1.5.2
DATABASE_URL=postgresql://prod-db-cluster:5432/myapp_prod
LOG_LEVEL=warn
API_REPLICAS=6
API_MEMORY_LIMIT=1g
API_PORT=8080
REDIS_VERSION=7
REDIS_MAX_MEMORY=1gb
```

## Step 3: Use Portainer's Environment Variable File Upload

For many variables, upload an `.env` file in Portainer's stack creation form instead of entering each variable manually. Store these files in your secrets manager and retrieve them during CI/CD.

## Step 4: Compose Override Files

For structural differences (not just value differences), use Docker Compose override files:

```yaml
# docker-compose.override.dev.yml — adds dev-only services
services:
  mailhog:
    image: mailhog/mailhog
    ports:
      - "8025:8025"   # Email testing UI, dev only
  
  api:
    # Mount source code for live reload in dev
    volumes:
      - ./src:/app/src:ro
    command: nodemon --watch /app/src server.js
```

In development, combine them:

```bash
docker compose -f docker-compose.yml -f docker-compose.override.dev.yml up
```

## Step 5: Portainer Custom Templates per Environment

Create separate Portainer App Templates for each environment tier with pre-filled variable values. Development templates include debug settings; production templates enforce minimum replicas and memory limits.

## Summary

Portainer's per-stack environment variable editor combined with parameterized Compose files creates a maintainable multi-environment deployment strategy. The same stack YAML runs everywhere; only the variable values change between environments. This eliminates config drift and makes environment promotion as simple as changing the IMAGE_TAG variable.
