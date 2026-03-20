# How to Configure Per-Container Updates with Watchtower in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Watchtower, Docker Labels, Automatic Updates, Container Management

Description: Learn how to use Watchtower labels to control update behavior on a per-container basis when deploying via Portainer, including enabling, disabling, and scheduling updates individually.

## Watchtower Label Overview

Watchtower supports several labels that control how it treats individual containers:

| Label | Type | Purpose |
|-------|------|---------|
| `com.centurylinklabs.watchtower.enable` | bool | Opt in/out of updates |
| `com.centurylinklabs.watchtower.stop-signal` | string | Custom stop signal |
| `com.centurylinklabs.watchtower.lifecycle.pre-update` | string | Command before update |
| `com.centurylinklabs.watchtower.lifecycle.post-update` | string | Command after update |
| `com.centurylinklabs.watchtower.depends-on` | string | Dependencies to stop first |
| `com.centurylinklabs.watchtower.scope` | string | Group containers by scope |

## Enabling Updates for Specific Containers

When Watchtower runs with `WATCHTOWER_LABEL_ENABLE=true`, only labeled containers are updated:

```yaml
# In Portainer: Stacks > Add Stack

services:
  frontend:
    image: myorg/frontend:latest
    labels:
      # This container will be auto-updated by Watchtower
      - "com.centurylinklabs.watchtower.enable=true"

  database:
    image: postgres:16
    # No label = Watchtower ignores this container
    volumes:
      - db_data:/var/lib/postgresql/data
```

## Excluding Containers from Updates

If Watchtower monitors all containers by default (no `LABEL_ENABLE`), use the exclude label:

```yaml
services:
  critical-service:
    image: myapp:v2.1.0    # Pinned version, do not update
    labels:
      # Explicitly exclude from Watchtower updates
      - "com.centurylinklabs.watchtower.enable=false"
```

## Custom Stop Signal

Some applications need a graceful shutdown signal:

```yaml
services:
  nginx:
    image: nginx:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      # Send SIGQUIT for graceful shutdown instead of SIGTERM
      - "com.centurylinklabs.watchtower.stop-signal=SIGQUIT"
```

## Pre-Update and Post-Update Hooks

Run commands before or after updating a container:

```yaml
services:
  app:
    image: myapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      # Drain connections before update
      - "com.centurylinklabs.watchtower.lifecycle.pre-update=echo 'Starting graceful drain...'"
      # Warm up cache after update
      - "com.centurylinklabs.watchtower.lifecycle.post-update=curl -s http://localhost:8080/warm-cache"
```

## Scoped Updates

Group containers and run separate Watchtower instances per scope:

```yaml
# Production containers
services:
  prod-app:
    image: myapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      - "com.centurylinklabs.watchtower.scope=production"

  staging-app:
    image: myapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      - "com.centurylinklabs.watchtower.scope=staging"
```

```yaml
# Separate Watchtower instances per scope
services:
  watchtower-prod:
    image: containrrr/watchtower:latest
    environment:
      - WATCHTOWER_SCOPE=production
      - WATCHTOWER_SCHEDULE=0 0 2 * * *    # 2 AM

  watchtower-staging:
    image: containrrr/watchtower:latest
    environment:
      - WATCHTOWER_SCOPE=staging
      - WATCHTOWER_SCHEDULE=0 0 * * * *    # Hourly
```

## Dependency Ordering

If container B must stop before container A updates:

```yaml
services:
  app:
    image: myapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      # Stop 'cache' container before updating 'app'
      - "com.centurylinklabs.watchtower.depends-on=cache"

  cache:
    image: redis:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

## Viewing Which Containers Watchtower Will Update

```bash
# Run Watchtower in monitor-only mode to see what would be updated
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --monitor-only --run-once
```

## Conclusion

Watchtower's label system gives you fine-grained control over which containers receive automatic updates, how they're stopped, and what runs before and after each update. When deploying via Portainer, simply add the appropriate labels to each service in your stack definition to establish the right update policy for each component.
