# How to Use Redis Docker Health Checks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Docker, Health Check

Description: Learn how to configure Docker health checks for Redis containers to ensure proper startup ordering, automatic restarts, and service dependency management.

---

Docker health checks allow the container runtime to determine whether your Redis container is actually ready to serve traffic - not just running as a process. This is critical for proper service dependencies and zero-downtime deployments.

## Basic Health Check

The simplest Redis health check uses `redis-cli PING`:

```dockerfile
FROM redis:7-alpine

HEALTHCHECK --interval=30s \
            --timeout=5s \
            --start-period=10s \
            --retries=3 \
  CMD redis-cli ping | grep -q PONG || exit 1
```

Health check parameters:

```text
--interval:     How often to run the check (default: 30s)
--timeout:      How long to wait for the check to complete (default: 30s)
--start-period: Grace period before failures count (default: 0s)
--retries:      Failed checks before marking unhealthy (default: 3)
```

## Health Check with Authentication

For password-protected Redis:

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD redis-cli -a "${REDIS_PASSWORD}" ping | grep -q PONG || exit 1
```

In Docker Compose:

```yaml
version: "3.8"
services:
  redis:
    image: redis:7-alpine
    environment:
      REDIS_PASSWORD: mysecretpassword
    command: redis-server --requirepass mysecretpassword
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "mysecretpassword", "PING"]
      interval: 10s
      timeout: 3s
      start_period: 10s
      retries: 3
```

## Advanced Health Check Script

A more thorough health check validates both connectivity and data operations:

```bash
#!/bin/bash
# redis-health.sh

# Check basic connectivity
if ! redis-cli -a "$REDIS_PASSWORD" PING | grep -q PONG; then
  echo "FAIL: Redis not responding to PING"
  exit 1
fi

# Check memory is not critically full
USED=$(redis-cli -a "$REDIS_PASSWORD" INFO memory | grep "^used_memory:" | cut -d: -f2 | tr -d '\r ')
MAX=$(redis-cli -a "$REDIS_PASSWORD" CONFIG GET maxmemory | tail -1 | tr -d '\r ')

if [ "$MAX" -gt 0 ] && [ "$((USED * 100 / MAX))" -gt 95 ]; then
  echo "WARN: Memory usage above 95%"
  exit 1
fi

# Verify write capability
redis-cli -a "$REDIS_PASSWORD" SET healthcheck "ok" EX 30 > /dev/null
VALUE=$(redis-cli -a "$REDIS_PASSWORD" GET healthcheck)
if [ "$VALUE" != "ok" ]; then
  echo "FAIL: Write/read test failed"
  exit 1
fi

exit 0
```

```dockerfile
COPY redis-health.sh /usr/local/bin/redis-health.sh
RUN chmod +x /usr/local/bin/redis-health.sh

HEALTHCHECK --interval=15s --timeout=5s --start-period=15s --retries=3 \
  CMD /usr/local/bin/redis-health.sh
```

## Using Health Checks for Service Dependencies

```yaml
version: "3.8"
services:
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "PING"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s

  app:
    image: myapp:latest
    depends_on:
      redis:
        condition: service_healthy  # waits for Redis to be healthy
    environment:
      REDIS_URL: redis://redis:6379
```

## Checking Health Status

```bash
# View health status
docker ps --filter "name=redis"
# NAMES    STATUS
# redis    Up 5 minutes (healthy)

# View health check logs
docker inspect redis --format='{{json .State.Health}}' | python3 -m json.tool

# Watch health transitions
docker events --filter "container=redis" --filter "event=health_status"
```

## Summary

Docker health checks for Redis use `redis-cli PING` as the foundation, extended with authentication flags and optional write/read tests for thorough validation. Configure appropriate `start_period` values to avoid false failures during startup, use `condition: service_healthy` in `depends_on` for proper service ordering, and monitor health status through `docker inspect` to debug startup issues.
