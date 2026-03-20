# How to Configure Startup Health Check Retries in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Health Checks, Startup, Retries

Description: Learn how to configure startup health check retries in Podman to control how many attempts are allowed during container initialization.

---

> Startup health check retries define how many times the startup probe can fail before the container is considered failed to start.

The number of startup retries directly determines the maximum startup window for your application. Combined with the startup interval, it defines the total time Podman will wait for an application to become ready before taking failure action.

---

## Setting Startup Health Check Retries

```bash
# Allow 30 retries during startup

podman run -d \
  --name startup-app \
  --health-startup-cmd "curl -f http://localhost:8080/ready || exit 1" \
  --health-startup-interval 5s \
  --health-startup-retries 30 \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  startup-app:latest

# Maximum startup time: 30 retries * 5s interval = 150 seconds
```

## Calculating Maximum Startup Time

```bash
# Formula: max_startup_time = retries * interval

# Fast-starting app: 10 retries * 2s = 20 seconds max
podman run -d --name fast-app \
  --health-startup-cmd "curl -f http://localhost:3000/ready || exit 1" \
  --health-startup-interval 2s \
  --health-startup-retries 10 \
  --health-cmd "curl -f http://localhost:3000/health || exit 1" \
  --health-interval 15s \
  fast-app:latest

# Slow-starting app: 60 retries * 10s = 600 seconds (10 minutes) max
podman run -d --name slow-app \
  --health-startup-cmd "curl -f http://localhost:8080/ready || exit 1" \
  --health-startup-interval 10s \
  --health-startup-retries 60 \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  slow-app:latest
```

## Retries for Different Application Types

```bash
# Node.js application (fast startup, ~5 seconds)
podman run -d --name node-app \
  --health-startup-cmd "wget -q --spider http://localhost:3000/ || exit 1" \
  --health-startup-interval 2s \
  --health-startup-retries 10 \
  --health-cmd "wget -q --spider http://localhost:3000/ || exit 1" \
  --health-interval 20s \
  node-app:latest

# Java Spring Boot (moderate startup, ~30-60 seconds)
podman run -d --name spring-app \
  --health-startup-cmd "curl -f http://localhost:8080/actuator/health || exit 1" \
  --health-startup-interval 5s \
  --health-startup-retries 24 \
  --health-cmd "curl -f http://localhost:8080/actuator/health || exit 1" \
  --health-interval 20s \
  spring-app:latest

# Database with large dataset import (slow startup, ~5 minutes)
podman run -d --name data-import-db \
  --health-startup-cmd "pg_isready -U postgres || exit 1" \
  --health-startup-interval 10s \
  --health-startup-retries 30 \
  --health-cmd "pg_isready -U postgres || exit 1" \
  --health-interval 30s \
  postgres-custom:latest
```

## Combining Retries with Failure Action

```bash
# Kill the container if startup retries are exhausted
podman run -d \
  --name strict-startup \
  --health-startup-cmd "curl -f http://localhost:8080/ready || exit 1" \
  --health-startup-interval 5s \
  --health-startup-retries 20 \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-on-failure kill \
  strict-startup-app:latest
```

## Summary

The `--health-startup-retries` flag controls how many times the startup health check can fail before the container is considered failed. The maximum startup window equals retries multiplied by the startup interval. Set this value based on your application's worst-case startup time, adding some margin for variable conditions like cold caches or slow network connections.
