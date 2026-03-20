# Best Practices for Stack Management in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Stacks, Docker Compose, Best Practices, DevOps, GitOps

Description: Establish best practices for creating, versioning, and maintaining Docker Compose stacks in Portainer to ensure reliable, repeatable deployments across your environments.

---

Stacks (Docker Compose definitions) are the primary deployment unit in Portainer. Managing them well - with proper versioning, environment separation, and secret management - is key to reliable deployments.

## 1. Use GitOps for Stack Definitions

Store your stack definitions in Git and deploy from repositories:

In Portainer, go to **Stacks > Add Stack > Repository** and provide:
- Git repository URL
- Branch name
- Path to the compose file

This means your stack definitions are:
- Version-controlled
- Peer-reviewed (via pull requests)
- Auditable (git history)
- Reproducible

## 2. Separate Configuration from the Compose File

Never hardcode environment-specific values in the compose file:

```yaml
# Bad - hardcoded values

services:
  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=postgres://prod-db:5432/mydb
      - API_KEY=abc123secret

# Good - use Portainer environment variables
services:
  app:
    image: myapp:${APP_VERSION:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - API_KEY=${API_KEY}
```

Set `APP_VERSION`, `DATABASE_URL`, and `API_KEY` as Portainer stack environment variables - different values per environment.

## 3. Pin Image Versions

Never use `latest` in production stacks:

```yaml
# Bad - unpredictable
image: nginx:latest

# Good - pinned version
image: nginx:1.25.4

# Even better - pinned by digest for full reproducibility
image: nginx@sha256:3b4b1b2b1b2b1b2b1b2b1b2b1b2b1b2b1b2b1b2b
```

Use a tool like Dependabot or Renovate to automate version updates via pull requests.

## 4. Add Health Checks to All Services

Portainer displays health status only for containers with health checks defined:

```yaml
services:
  api:
    image: myapi:1.2.3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s   # Allow extra time for startup
```

## 5. Set Resource Limits

Always define resource limits to prevent runaway containers:

```yaml
services:
  worker:
    image: myworker:1.2.3
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.50"
        reservations:
          memory: 128M
          cpus: "0.10"
```

## 6. Use Restart Policies Appropriately

```yaml
# Production services - always restart
restart: unless-stopped

# One-off jobs - don't restart
restart: "no"

# Services that should restart only on failure
restart: on-failure:3  # Max 3 retries
```

## 7. Named Volumes Over Bind Mounts

Use named volumes for portability:

```yaml
# Prefer named volumes
volumes:
  db-data:/var/lib/postgresql/data

# Over bind mounts (host-dependent)
# - /opt/myapp/db-data:/var/lib/postgresql/data
```

Named volumes are managed by Docker and portable across environments.

## 8. Stack Naming Convention

Use consistent stack names:

```text
<app>-<tier>

Examples:
- wordpress-production
- api-staging
- monitoring-shared
- database-production
```

## Summary

Well-managed Portainer stacks are version-controlled, use environment variables for configuration, pin image versions, include health checks, and set resource limits. These practices prevent the common problems of "works on staging but not on production" and unplanned resource exhaustion.
