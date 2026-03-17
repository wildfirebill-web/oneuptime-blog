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
  production-api:latest

# This configuration:
# - Waits 45 seconds before counting failures
# - Checks every 20 seconds with a 10-second timeout
# - Restarts after 3 consecutive failures
#
# Note: Do not combine --health-on-failure restart with --restart.
# The Podman documentation advises against using both together.
# When running inside a systemd unit, use the kill or stop action
# instead and let systemd handle restarts.
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

When using `--health-on-failure restart` outside of systemd, the container will keep restarting as long as it becomes unhealthy. To prevent infinite loops, use higher `--health-retries` values and longer intervals:

```bash
# Use higher retries and longer intervals to limit restart frequency
podman run -d \
  --name safe-restart-app \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 60s \
  --health-retries 5 \
  --health-start-period 30s \
  --health-on-failure restart \
  my-app:latest

# For systemd-managed containers, prefer --health-on-failure kill
# and let systemd's Restart=on-failure handle recovery with limits
```

## Summary

Configuring health checks with `--health-on-failure restart` creates self-healing containers that automatically recover from application failures. Use `--health-start-period` to give applications time to initialize. When running inside a systemd unit, prefer the `kill` or `stop` action and let systemd handle restarts instead of using the `restart` action. Do not combine `--health-on-failure restart` with the `--restart` flag.
