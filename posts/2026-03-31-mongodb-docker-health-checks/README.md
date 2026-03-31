# How to Use Docker Health Checks for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Docker, Health Check, Container, Reliability

Description: Configure Docker health checks for MongoDB containers to ensure dependent services only start after the database is fully ready and accepting connections.

---

## Overview

Docker's `HEALTHCHECK` instruction lets the container runtime periodically verify that a service is actually running correctly, not just that the process is alive. For MongoDB, a health check should verify that the server accepts connections and responds to commands - ensuring that dependent application containers start only after MongoDB is ready.

## Basic Health Check with mongosh

```yaml
version: "3.8"
services:
  mongo:
    image: mongo:7.0
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping').ok", "--quiet"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

volumes:
  mongo_data:
```

## Authenticated Health Check

When MongoDB requires authentication, pass credentials in the health check command.

```yaml
healthcheck:
  test:
    - CMD
    - mongosh
    - "--username"
    - "admin"
    - "--password"
    - "secret"
    - "--authenticationDatabase"
    - "admin"
    - "--eval"
    - "db.adminCommand('ping').ok"
    - "--quiet"
  interval: 15s
  timeout: 10s
  retries: 5
  start_period: 40s
```

## Replica Set Health Check

For replica set deployments, verify that the node has reached PRIMARY or SECONDARY status.

```yaml
healthcheck:
  test:
    - CMD
    - mongosh
    - "--eval"
    - >
      var status = rs.status();
      if (status.myState !== 1 && status.myState !== 2) quit(1);
    - "--quiet"
  interval: 20s
  timeout: 10s
  retries: 10
  start_period: 60s
```

## Dependent Service Using condition: service_healthy

Use `depends_on` with the `service_healthy` condition to block application startup until MongoDB passes its health check.

```yaml
services:
  mongo:
    image: mongo:7.0
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping').ok", "--quiet"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  api:
    build: ./api
    depends_on:
      mongo:
        condition: service_healthy
    environment:
      MONGODB_URI: mongodb://admin:secret@mongo:27017/myapp?authSource=admin
```

## Health Check in a Dockerfile

Add a health check directly to the image for portability outside Docker Compose.

```dockerfile
FROM mongo:7.0

HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=5 \
  CMD mongosh --eval "db.adminCommand('ping').ok" --quiet || exit 1
```

## Checking Health Status

```bash
# View health status
docker inspect mongo-container --format='{{.State.Health.Status}}'

# Watch health check output
docker inspect mongo-container \
  --format='{{range .State.Health.Log}}{{.ExitCode}} {{.Output}}{{end}}'

# In Docker Compose
docker compose ps
```

## Health Check Tuning Guidelines

| Parameter | Recommended Value | Reason |
|-----------|-------------------|--------|
| `interval` | 10s - 15s | Frequent enough to detect failure quickly |
| `timeout` | 5s - 10s | MongoDB ping is fast; fail early |
| `retries` | 5 | Allow for startup time variance |
| `start_period` | 30s - 60s | Skip failures during initial startup |

## Summary

Docker health checks for MongoDB use `mongosh --eval "db.adminCommand('ping').ok"` to confirm the server is accepting connections. Set a `start_period` long enough for MongoDB to initialize on first boot, use `condition: service_healthy` in `depends_on` to block dependent services, and tune `interval` and `retries` based on your deployment environment.
