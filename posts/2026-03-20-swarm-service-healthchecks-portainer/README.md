# How to Configure Docker Swarm Service Health Checks with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Health Check, Container Management, DevOps

Description: Learn how to configure health checks for Docker Swarm services using Portainer, ensuring unhealthy containers are automatically detected and replaced.

## Introduction

Health checks allow Docker Swarm to automatically detect when a service container is not functioning correctly and replace it. Portainer provides a UI to configure health checks when deploying or updating Swarm services.

## Configuring Health Checks via Portainer UI

1. Open Portainer and navigate to **Services**
2. Click **Add Service** or edit an existing service
3. Scroll to the **Service details** section
4. Under **Health check**, configure:
   - **Command**: the test command (e.g., `curl -f http://localhost/health`)
   - **Interval**: how often to run the check (e.g., `30s`)
   - **Timeout**: how long to wait for a response (e.g., `10s`)
   - **Retries**: how many failures before marking unhealthy (e.g., `3`)
   - **Start period**: grace period for startup (e.g., `60s`)

## Defining Health Checks in a Stack File

Deploy via Portainer using a stack YAML with health check defined:

```yaml
version: "3.8"

services:
  web:
    image: nginx:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    ports:
      - "80:80"
```

## Health Check for a Custom Application

```yaml
services:
  api:
    image: myapp:latest
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
```

## Monitoring Health Status in Portainer

After deploying:

1. Go to **Services** and click your service
2. Click on a task to view its health check status
3. Health states shown: `healthy`, `unhealthy`, `starting`

## Viewing Health Check Logs

In Portainer, navigate to the container's logs tab. Alternatively via CLI:

```bash
docker inspect --format='{{json .State.Health}}' <container_id>
```

## Swarm Rolling Updates with Health Checks

Swarm respects health checks during rolling updates. If a new task becomes unhealthy, the update stops and can roll back:

```yaml
deploy:
  update_config:
    failure_action: rollback
    monitor: 30s
```

## Conclusion

Configuring health checks for Docker Swarm services in Portainer ensures your applications are continuously monitored and automatically recovered from failures. By combining health checks with Swarm's update policies, you can achieve zero-downtime deployments with automatic rollback on failure.
