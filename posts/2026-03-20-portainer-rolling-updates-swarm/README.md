# How to Implement Rolling Updates with Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Rolling Updates, Zero Downtime, Deployment, Services

Description: Learn how to implement zero-downtime rolling updates for Docker Swarm services via Portainer, controlling update parallelism, delay, and failure actions.

---

Rolling updates replace service replicas one at a time (or in small batches), ensuring the application remains available throughout the deployment. Docker Swarm supports this natively via the `update_config` section of service definitions.

## Configuring Rolling Updates in a Stack

The `update_config` block controls how Swarm applies image changes:

```yaml
version: "3.8"

services:
  api:
    image: myregistry.example.com/my-app:${IMAGE_TAG:-latest}
    deploy:
      replicas: 4
      update_config:
        parallelism: 1        # Update 1 replica at a time
        delay: 15s            # Wait 15s between each replica update
        failure_action: rollback   # Automatically rollback on failure
        monitor: 30s          # Observe for 30s before marking success
        max_failure_ratio: 0.25    # Allow up to 25% replicas to fail before rollback
        order: start-first    # Start new replica before stopping old one
      rollback_config:
        parallelism: 2
        delay: 5s
        failure_action: pause
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
```

The `order: start-first` option ensures the new replica is healthy before the old one is removed, giving true zero-downtime behavior.

## Triggering a Rolling Update via Portainer

Update the image tag in the stack and click **Update the stack** in Portainer. Swarm automatically performs the rolling update using the configured `update_config`.

Alternatively, trigger from the CLI:

```bash
# Update the service image

docker service update \
  --image myregistry.example.com/my-app:v1.5.0 \
  --update-parallelism 1 \
  --update-delay 15s \
  mystack_api

# Watch the update progress
docker service ps mystack_api --no-trunc
```

## Monitoring Rolling Update Progress

Track which replicas are running the new image:

```bash
# Watch update status in real time
watch -n 2 'docker service ps mystack_api --format "{{.Name}}\t{{.Image}}\t{{.CurrentState}}"'

# Check overall service update status
docker service inspect mystack_api --format '{{.UpdateStatus.State}}'
# Returns: updating, paused, completed, or rollback_started
```

## Automatic Rollback on Health Check Failure

With `failure_action: rollback` and a health check configured, Swarm automatically rolls back if newly started replicas fail their health checks within the `monitor` window:

```bash
# If a rollback is triggered, observe it
docker service inspect mystack_api --format '{{.UpdateStatus.Message}}'

# Manually trigger a rollback if needed
docker service rollback mystack_api
```

## Blue-Green Equivalent with Swarm Labels

For a true blue-green switch with Swarm, use separate services and a label-based router:

```yaml
  api-green:
    image: myregistry.example.com/my-app:v1.5.0
    deploy:
      labels:
        - "traefik.http.routers.api.service=api-green"
    # Start in parallel with blue, then remove blue after verification

  api-blue:
    image: myregistry.example.com/my-app:v1.4.0
    deploy:
      replicas: 0   # Scale down after green is verified
```

## Rolling Update Best Practices

| Setting | Recommendation |
|---------|----------------|
| `parallelism` | 1 for critical services, up to 25% of replicas for fast deploys |
| `delay` | At least 10-30s to allow health checks to run |
| `failure_action` | `rollback` for production, `pause` for debugging |
| `monitor` | At least 2x your health check interval |
| `order` | `start-first` for stateless services |
