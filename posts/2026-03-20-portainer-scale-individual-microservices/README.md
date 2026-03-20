# How to Scale Individual Microservices in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Scaling, Microservices, Docker Swarm, Load Balancing, Docker Compose

Description: Learn how to scale individual microservice containers in Portainer using Docker Compose replica settings and Docker Swarm service scaling.

---

Portainer provides UI-based scaling for services in Docker Swarm mode and Compose stacks. This guide covers both approaches for scaling individual microservices independently.

## Scaling in Docker Swarm via Portainer

For Swarm services, Portainer provides a direct scaling UI:

1. Go to **Swarm > Services**.
2. Click the service you want to scale.
3. Change the **Replicas** number.
4. Click **Update**.

Swarm automatically distributes the new replicas across nodes.

## Scaling via Stack Compose File

Update the `deploy.replicas` count in the stack YAML:

```yaml
version: "3.8"
services:
  user-service:
    image: user-service:latest
    deploy:
      replicas: 3         # Scale to 3 instances
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
      restart_policy:
        condition: on-failure
        max_attempts: 3
      update_config:
        parallelism: 1    # Update one replica at a time
        delay: 10s

  # Heavy service needs more replicas
  api-gateway:
    image: nginx:alpine
    deploy:
      replicas: 5
```

## Scaling via Docker CLI

```bash
# Scale a Swarm service from the command line
docker service scale mystack_user-service=5

# Verify the scaling
docker service ps mystack_user-service
```

## Horizontal Scaling with Load Balancing

Docker Swarm automatically load-balances requests across replicas using the virtual IP (VIP) routing mesh. No additional load balancer configuration is needed for internal service-to-service calls:

```javascript
// This request is automatically load-balanced to one of the user-service replicas
const response = await fetch('http://user-service:3001/users');
```

## Auto-Scaling Scripts

Docker Swarm does not have built-in auto-scaling, but you can implement it with a script:

```bash
#!/bin/bash
# scale-by-cpu.sh - Scale up if average CPU > 80%

CPU=$(docker stats --no-stream --format "{{.CPUPerc}}" $(docker service ps mystack_user-service -q --no-trunc | head -1 | awk '{print $1}') 2>/dev/null | tr -d '%')

if (( $(echo "$CPU > 80" | bc -l) )); then
  CURRENT=$(docker service inspect mystack_user-service --format '{{.Spec.Mode.Replicated.Replicas}}')
  docker service scale mystack_user-service=$((CURRENT + 1))
  echo "Scaled up to $((CURRENT + 1)) replicas (CPU was ${CPU}%)"
fi
```

## Monitoring Replicas

In Portainer, the **Services** view shows the current and desired replica count. OneUptime can monitor the service endpoint and alert if all replicas are unhealthy.
