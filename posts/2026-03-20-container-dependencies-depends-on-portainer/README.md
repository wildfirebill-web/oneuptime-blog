# How to Set Up Container Dependencies (depends_on) in Portainer Stacks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Compose, depends_on, Service Ordering, DevOps, Reliability

Description: Configure service startup ordering and dependency health checks using depends_on in Portainer Docker Compose stacks to ensure services start in the correct sequence with proper health verification.

---

The `depends_on` directive in Docker Compose controls the startup order of services in a Portainer stack. Without it, services start simultaneously, which can cause application startup failures when a service tries to connect to a database that isn't ready yet.

## Basic depends_on (Start Order Only)

The simplest form ensures services start in order but doesn't wait for them to be healthy:

```yaml
version: "3.8"
services:
  app:
    image: myapp:1.2.3
    depends_on:
      - database    # database container starts before app
      - redis       # redis container starts before app

  database:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine
```

**Problem**: The app container starts as soon as `database` and `redis` containers start, not when they're ready to accept connections.

## Health-Based depends_on (Recommended)

Use condition checks to wait for actual service readiness:

```yaml
version: "3.8"
services:
  app:
    image: myapp:1.2.3
    depends_on:
      database:
        condition: service_healthy    # Wait for healthy health check
      redis:
        condition: service_healthy

  database:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=db_password
    healthcheck:
      # pg_isready returns 0 when the database accepts connections
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s    # Give PostgreSQL time to initialize

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
```

## Dependency Conditions Reference

| Condition | Meaning |
|-----------|---------|
| `service_started` | Container has started (default) |
| `service_healthy` | Container health check is passing |
| `service_completed_successfully` | Container exited with code 0 (for init containers) |

## Real-World Multi-Service Stack

A typical web application with proper dependency ordering:

```yaml
version: "3.8"
services:
  nginx:
    image: nginx:1.25
    ports:
      - "80:80"
    depends_on:
      app:
        condition: service_healthy

  app:
    image: myapp:1.2.3
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  database:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

volumes:
  db-data:
```

## Application-Level Retry Logic

`depends_on` helps but isn't sufficient for all cases. Add retry logic in your application:

```python
# db_connect.py — retry logic in the application
import psycopg2
import time
import os

def connect_with_retry(max_retries=10, delay=2):
    """Retry database connection with exponential backoff."""
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(os.environ["DATABASE_URL"])
            print(f"Connected to database on attempt {attempt + 1}")
            return conn
        except psycopg2.OperationalError as e:
            if attempt == max_retries - 1:
                raise
            wait = delay * (2 ** attempt)
            print(f"Database not ready (attempt {attempt + 1}), retrying in {wait}s...")
            time.sleep(wait)
```

## Summary

Health-based `depends_on` conditions are the right way to handle service startup ordering in Portainer stacks. Always define health checks on your database and cache services, and use `service_healthy` conditions in your application service's `depends_on` block. Combine this with application-level retry logic for maximum reliability.
