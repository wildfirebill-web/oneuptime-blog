# How to Set Up Health Checks for Microservices in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Health Checks, Microservices, Monitoring, Reliability

Description: Configure Docker health checks for microservices to enable automatic restart, load balancer awareness, and deployment safety with Portainer.

## Introduction

Docker health checks automatically monitor container health and inform orchestrators (Swarm, Kubernetes) and load balancers whether a container is ready to receive traffic. A container can be running but unhealthy (e.g., application crashed but process is still alive). Proper health checks are critical for zero-downtime deployments. This guide covers implementing comprehensive health checks managed through Portainer.

## Step 1: Basic Health Check in Dockerfile

```dockerfile
# Dockerfile - Built-in health check

FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

EXPOSE 8000

# Health check definition
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=30s \
            --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Step 2: Health Check in docker-compose.yml

```yaml
# docker-compose.yml - Health checks for all services
version: "3.8"

networks:
  app_network:
    driver: bridge

services:
  # Web API
  api:
    image: myapp/api:latest
    healthcheck:
      # Test command
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      # Check every 30 seconds
      interval: 30s
      # Consider unhealthy after 10s timeout
      timeout: 10s
      # Wait 30s before first check (startup period)
      start_period: 30s
      # Mark unhealthy after 3 consecutive failures
      retries: 3
    restart: unless-stopped
    networks:
      - app_network

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppass
    networks:
      - app_network

  # MySQL
  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "--password=${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
    networks:
      - app_network

  # MongoDB
  mongodb:
    image: mongo:7.0
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')", "--quiet"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app_network

  # Redis
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app_network

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management-alpine
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - app_network

  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.12.0
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cat/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    environment:
      - discovery.type=single-node
    networks:
      - app_network

  # Service B waits for Service A to be healthy
  service_b:
    image: service-b:latest
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      api:
        condition: service_healthy
    networks:
      - app_network
```

## Step 3: Implement Health Check Endpoints

```python
# Python FastAPI - Comprehensive health endpoint
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse
import asyncpg
import redis.asyncio as redis
import os

app = FastAPI()

@app.get("/health")
async def health_check():
    """Basic liveness check - is the application running?"""
    return {"status": "healthy"}

@app.get("/health/ready")
async def readiness_check():
    """Readiness check - is the app ready to serve traffic?"""
    checks = {}
    overall_healthy = True

    # Database check
    try:
        conn = await asyncpg.connect(os.environ["DATABASE_URL"])
        await conn.execute("SELECT 1")
        await conn.close()
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {str(e)}"
        overall_healthy = False

    # Redis check
    try:
        r = redis.from_url(os.environ["REDIS_URL"])
        await r.ping()
        await r.aclose()
        checks["redis"] = "healthy"
    except Exception as e:
        checks["redis"] = f"unhealthy: {str(e)}"
        overall_healthy = False

    status_code = status.HTTP_200_OK if overall_healthy else status.HTTP_503_SERVICE_UNAVAILABLE

    return JSONResponse(
        status_code=status_code,
        content={
            "status": "healthy" if overall_healthy else "unhealthy",
            "checks": checks
        }
    )

@app.get("/health/live")
async def liveness_check():
    """Simple liveness check for Kubernetes/Docker"""
    return {"status": "alive"}
```

```go
// Go - Health check endpoint
package health

import (
    "database/sql"
    "encoding/json"
    "net/http"
)

type HealthStatus struct {
    Status string            `json:"status"`
    Checks map[string]string `json:"checks"`
}

func Handler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        checks := map[string]string{}
        healthy := true

        // Database check
        if err := db.PingContext(r.Context()); err != nil {
            checks["database"] = "unhealthy: " + err.Error()
            healthy = false
        } else {
            checks["database"] = "healthy"
        }

        status := HealthStatus{Checks: checks}
        if healthy {
            status.Status = "healthy"
            w.WriteHeader(http.StatusOK)
        } else {
            status.Status = "unhealthy"
            w.WriteHeader(http.StatusServiceUnavailable)
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(status)
    }
}
```

## Step 4: View Health Status in Portainer

Portainer displays health status in multiple places:
- **Containers** list: Shows a green/yellow/red dot for health status
- **Container details**: Shows health check history and last output
- **Stacks**: Shows overall stack health

```bash
# Check health status via Docker CLI
docker inspect --format='{{.State.Health}}' mycontainer

# View health check log
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' mycontainer

# Force a health check immediately
docker exec mycontainer curl -f http://localhost:8000/health
```

## Step 5: Health Checks in Docker Swarm

```yaml
# swarm-stack.yml - Health checks for Swarm services
services:
  api:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      replicas: 3
      update_config:
        # Wait for health check before updating next replica
        order: start-first
        failure_action: rollback
      rollback_config:
        parallelism: 1
```

## Conclusion

Health checks are fundamental to reliable microservice deployments. With proper health checks in place, Docker will automatically restart unhealthy containers, Traefik will stop routing traffic to failing instances, and Swarm will restart replicas that fail health checks. Portainer's container dashboard shows health status at a glance, making it easy to spot problematic services before they impact users. Use the readiness/liveness pattern to distinguish between "app is starting" and "app is broken."
