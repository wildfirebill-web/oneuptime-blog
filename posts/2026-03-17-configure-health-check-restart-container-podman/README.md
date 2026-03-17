# How to Configure Health Check to Restart a Container in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Health Checks, Auto-Restart, Self-Healing

Description: Learn how to configure Podman health checks to automatically restart containers when they become unhealthy.

---

> Configuring health checks to restart unhealthy containers creates a self-healing system that recovers from failures without manual intervention.

Automatic restarts on health check failure are one of the most practical features for running production containers. By combining health checks with the restart-on-failure action, you can build resilient services that recover automatically from transient application issues.

---

## Basic Restart on Health Failure

```bash
# Restart the container automatically when health check fails
podman run -d \
  --name auto-restart-app \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 15s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-on-failure restart \
  my-app:latest
```

## Full Configuration for Production

```bash
# Production-ready self-healing container
podman run -d \
  --name production-api \
  --health-cmd "curl -sf http://localhost:8080/health || exit 1" \
  --health-interval 20s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 45s \
  --health-on-failure restart \
  --restart on-failure:10 \
  production-api:latest

# This configuration:
# - Waits 45 seconds before counting failures
# - Checks every 20 seconds with a 10-second timeout
# - Restarts after 3 consecutive failures
# - Stops restarting after 10 restart attempts
```

## Health Check with Restart for Different Services

```bash
# Web server with HTTP health check and auto-restart
podman run -d --name nginx-app \
  --health-cmd "curl -f http://localhost:80/ || exit 1" \
  --health-interval 30s \
  --health-retries 3 \
  --health-on-failure restart \
  nginx:latest

# Database with connection health check and auto-restart
podman run -d --name postgres-db \
  --health-cmd "pg_isready -U postgres || exit 1" \
  --health-interval 30s \
  --health-retries 5 \
  --health-start-period 30s \
  --health-on-failure restart \
  postgres:15

# Redis with ping health check and auto-restart
podman run -d --name redis-cache \
  --health-cmd "redis-cli ping | grep -q PONG || exit 1" \
  --health-interval 15s \
  --health-retries 3 \
  --health-on-failure restart \
  redis:7
```

## Monitoring Restart Events

```bash
# Watch for restart events triggered by health check failures
podman events --filter event=restart

# Check how many times a container has been restarted
podman inspect --format='{{.RestartCount}}' auto-restart-app
```

## Avoiding Restart Loops

```bash
# Combine restart limit with health checks to prevent infinite loops
podman run -d \
  --name safe-restart-app \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-retries 5 \
  --health-on-failure restart \
  --restart on-failure:5 \
  my-app:latest

# After 5 restart attempts, the container will stop restarting
```

## Summary

Configuring health checks with `--health-on-failure restart` creates self-healing containers that automatically recover from application failures. Combine this with `--restart on-failure:N` to prevent infinite restart loops, and use `--health-start-period` to give applications time to initialize. This approach provides reliable service recovery without requiring external orchestration.
