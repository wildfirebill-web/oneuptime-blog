# How to Configure Container Health Checks in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Health Check, Monitoring, Reliability, DevOps

Description: Define and configure Docker container health checks in Portainer stacks to enable automatic restart on failure, proper depends_on ordering, and health status visibility in the Portainer dashboard.

---

Health checks tell Docker (and Portainer) whether a container is actually functioning correctly, not just running. Portainer displays health status in the container list and uses it for `depends_on` conditions.

## Health Check States

A container with a health check can be in four states:

- **starting** - health check hasn't run yet (within `start_period`)
- **healthy** - last N checks passed
- **unhealthy** - last N checks failed (triggers restart if `restart: always`)
- **none** - no health check defined

## Defining Health Checks in Portainer Stacks

### HTTP Health Check

For web services:

```yaml
services:
  webapp:
    image: myapp:1.2.3
    healthcheck:
      # HTTP endpoint that returns 2xx for healthy
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s        # Check every 30 seconds
      timeout: 10s         # Fail if no response in 10 seconds
      retries: 3           # Mark unhealthy after 3 consecutive failures
      start_period: 40s    # Grace period during startup
```

### TCP Port Check

For databases and services without HTTP endpoints:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      # pg_isready returns 0 when PostgreSQL is accepting connections
      test: ["CMD-SHELL", "pg_isready -h localhost -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

### Redis Health Check

```yaml
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

### Custom Health Check Script

For complex checks:

```yaml
  app:
    image: myapp:1.2.3
    healthcheck:
      test: ["CMD", "/usr/local/bin/healthcheck.sh"]
      interval: 30s
      timeout: 10s
      retries: 3
```

```bash
#!/bin/sh
# healthcheck.sh - inside the container

# Returns 0 for healthy, 1 for unhealthy

# Check HTTP endpoint
curl -sf http://localhost:8080/health || exit 1

# Check required file exists
test -f /app/config/runtime.json || exit 1

# Check database connectivity
pg_isready -h postgres || exit 1

exit 0
```

## Health Check Parameters Explained

| Parameter | Default | Description |
|-----------|---------|-------------|
| `interval` | 30s | Time between checks |
| `timeout` | 30s | Max time for check to complete |
| `retries` | 3 | Failures before marking unhealthy |
| `start_period` | 0s | Grace period before checks count |

Set `start_period` to at least the expected startup time of your application. During `start_period`, failed checks are not counted toward `retries`.

## Viewing Health Status in Portainer

In Portainer's **Containers** list:
- A green dot = `healthy`
- A yellow dot = `starting` or `unhealthy`
- Gray = no health check defined

Click a container to see detailed health check history in the **Inspect** tab.

## Triggering Restarts on Unhealthy

Combine health checks with restart policies to auto-recover:

```yaml
services:
  webapp:
    image: myapp:1.2.3
    restart: unless-stopped    # Restart on exit or unhealthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      retries: 3
```

When the container becomes `unhealthy`, Docker automatically restarts it (similar to Kubernetes liveness probes).

## Summary

Container health checks are a fundamental reliability feature. Define them for all production services in your Portainer stacks - especially databases and API services. Portainer's health status display gives immediate visibility into container health, and automatic restarts on unhealthy state reduce manual intervention for recoverable failures.
