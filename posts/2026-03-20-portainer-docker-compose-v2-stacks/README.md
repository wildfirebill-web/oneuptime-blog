# How to Use Docker Compose v2 Syntax in Portainer Stacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Docker Compose, Stack, DevOps

Description: Learn how to leverage Docker Compose v2 syntax features in your Portainer stacks for modern container deployments.

## Introduction

Docker Compose v2 introduced significant improvements over v1, including better performance, new features, and tighter integration with Docker Engine. Portainer supports Docker Compose v2 syntax in its stacks feature, allowing you to use all modern Compose capabilities. This guide covers the key v2 syntax features and how to use them in Portainer.

## Prerequisites

- Portainer CE or BE 2.9+
- Docker Engine 20.10+ with Compose plugin installed
- Familiarity with basic Docker Compose concepts

## Compose v2 vs v1: Key Differences

| Feature | Compose v1 | Compose v2 |
|---------|-----------|-----------|
| CLI command | `docker-compose` | `docker compose` |
| Installation | Standalone Python binary | Docker plugin |
| Profile support | No | Yes |
| `depends_on` conditions | Basic | Health check conditions |
| `include` directive | No | Yes |
| `extend` support | Limited | Improved |

## Step 1: Remove the Version Key (Optional)

In Compose v2, the top-level `version` key is optional and ignored. You can omit it for cleaner files:

```yaml
# Compose v2 - no version key needed

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
```

## Step 2: Use `depends_on` with Health Checks

Compose v2 supports condition-based dependency management:

```yaml
services:
  api:
    image: myapi:latest
    depends_on:
      database:
        condition: service_healthy   # Wait for DB health check to pass
      cache:
        condition: service_started   # Just wait for container to start

  database:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s   # Grace period for initial startup

  cache:
    image: redis:7-alpine
```

## Step 3: Use Profiles to Group Services

Profiles let you conditionally start services:

```yaml
services:
  web:
    image: nginx:alpine
    # No profile - always starts

  api:
    image: myapi:latest
    # No profile - always starts

  debug-tools:
    image: nicolaka/netshoot
    profiles:
      - debug   # Only starts when 'debug' profile is active

  monitoring:
    image: prom/prometheus
    profiles:
      - monitoring   # Only starts with 'monitoring' profile
```

In Portainer, set the `COMPOSE_PROFILES` environment variable to activate profiles.

## Step 4: Use the `include` Directive

Break large Compose files into smaller, reusable fragments:

```yaml
# Main docker-compose.yml
include:
  - path: ./compose/database.yml    # Include database services
  - path: ./compose/monitoring.yml  # Include monitoring services

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
```

```yaml
# compose/database.yml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
    volumes:
      - pg-data:/var/lib/postgresql/data

volumes:
  pg-data:
```

## Step 5: Use Build Secrets

Compose v2 supports build-time secrets that don't persist in image layers:

```yaml
services:
  app:
    build:
      context: .
      secrets:
        - github_token   # Available during build, not in final image
    image: myapp:latest

secrets:
  github_token:
    environment: GITHUB_TOKEN   # Read from environment variable
```

## Step 6: Use `develop` for Watch Mode

The `develop` key enables file watching and automatic rebuilds:

```yaml
services:
  frontend:
    image: node:18
    command: npm run dev
    develop:
      watch:
        - action: sync           # Sync files without rebuild
          path: ./src
          target: /app/src
        - action: rebuild        # Rebuild image on Dockerfile change
          path: Dockerfile
```

## Step 7: Advanced Network Configuration

```yaml
networks:
  frontend-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24   # Custom subnet

  backend-net:
    driver: bridge
    internal: true   # No external connectivity

services:
  web:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net   # Connected to both networks

  database:
    networks:
      - backend-net   # Only on internal network
```

## Deploying in Portainer

1. Navigate to **Stacks > Add stack**
2. Paste your v2 Compose file in the web editor
3. Add any required environment variables
4. Click **Deploy the stack**

Portainer automatically uses the Docker Compose v2 plugin when deploying on standalone Docker environments.

## Troubleshooting

- **"version is obsolete" warning** - safe to ignore or remove the `version` key
- **Profile not activating** - set `COMPOSE_PROFILES=profile-name` in stack environment variables
- **`include` not found** - ensure referenced files exist in the repository when using Git-based stacks

## Conclusion

Docker Compose v2 brings powerful new capabilities to Portainer stacks. Features like health-check-based dependencies, profiles, and the `include` directive make it easier to manage complex applications. By adopting v2 syntax, your stacks become more robust, modular, and production-ready.
