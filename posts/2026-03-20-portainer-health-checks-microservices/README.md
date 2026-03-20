# How to Set Up Health Checks for Microservices in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Health Checks, Microservices, Docker, Docker Compose, Reliability

Description: Learn how to configure Docker health checks for microservices in Portainer to enable automatic restart on failure and dependency-aware startup ordering.

---

Docker health checks tell Portainer (and Docker Swarm) whether a container is functioning correctly. Proper health checks enable automatic restarts on failure, dependency ordering on startup, and accurate status in the Portainer UI.

## Defining Health Checks in Compose

```yaml
version: "3.8"
services:
  api:
    image: myapi:latest
    healthcheck:
      # Command to run inside the container
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s      # How often to check
      timeout: 10s       # Max time to wait for response
      retries: 3         # Failures before marking unhealthy
      start_period: 30s  # Grace period for slow startup
```

## Health Check Patterns by Service Type

**HTTP API:**
```yaml
test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

**TCP port:**
```yaml
test: ["CMD-SHELL", "nc -z localhost 5432 || exit 1"]
```

**Database (PostgreSQL):**
```yaml
test: ["CMD-SHELL", "pg_isready -U postgres"]
```

**Database (MySQL):**
```yaml
test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
```

**Redis:**
```yaml
test: ["CMD", "redis-cli", "ping"]
```

## Dependency Health Ordering

Use health check conditions to start services only when dependencies are healthy:

```yaml
version: "3.8"
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      retries: 10

  api:
    image: myapi:latest
    depends_on:
      db:
        # Wait for db to be healthy before starting api
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      retries: 3
```

## Viewing Health Status in Portainer

In Portainer's container list, containers show their health status:

- Green circle: **Healthy**
- Yellow circle: **Starting** (within `start_period`)
- Red circle: **Unhealthy** (exceeded retries)
- Grey circle: **No health check**

## Auto-Restart on Unhealthy

Combine health checks with restart policies for automatic recovery:

```yaml
services:
  api:
    image: myapi:latest
    restart: unless-stopped    # Restart if container exits or becomes unhealthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      retries: 3
```

Docker will restart the container when health check fails after `retries` consecutive failures.
