# How to Deploy Redis with Docker Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Docker, Swarm, Deployment, Container

Description: Deploy a production-ready Redis instance on Docker Swarm with persistent storage, health checks, and resource limits using a Swarm stack file.

---

Docker Swarm provides a simpler alternative to Kubernetes for teams that need container orchestration without the operational complexity. Deploying Redis on Swarm gives you rolling updates, health checks, and overlay networking out of the box.

## Stack File for Redis

Create a `redis-stack.yml` file that defines the Redis service with constraints and resource limits:

```yaml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    ports:
      - target: 6379
        published: 6379
        protocol: tcp
        mode: host
    volumes:
      - redis_data:/data
    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          memory: 768M
          cpus: "0.5"
        reservations:
          memory: 256M
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    networks:
      - redis_net

volumes:
  redis_data:
    driver: local

networks:
  redis_net:
    driver: overlay
    attachable: true
```

## Deploying the Stack

Initialize Swarm on your manager node, then deploy:

```bash
# Initialize Docker Swarm (if not already done)
docker swarm init

# Set your Redis password as a Swarm secret or environment variable
export REDIS_PASSWORD="your-strong-password-here"

# Deploy the stack
docker stack deploy -c redis-stack.yml redis

# Verify the service is running
docker service ls
docker service ps redis_redis
```

## Connecting Other Services

Attach application services to the same overlay network:

```yaml
services:
  app:
    image: myapp:latest
    networks:
      - redis_redis_net
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
    deploy:
      replicas: 3
```

Note: when referencing a network created by another stack, use `{stack}_{network}` naming.

## Monitoring Redis Health

Check service health and logs from the Swarm manager:

```bash
# View service health
docker service inspect --pretty redis_redis

# Check logs
docker service logs redis_redis --tail 50 --follow

# Run redis-cli inside the container
docker exec -it $(docker ps -q -f name=redis_redis) \
  redis-cli -a $REDIS_PASSWORD info server
```

## Scaling Considerations

Redis is stateful and single-threaded per instance, so horizontal scaling via Swarm replicas is not straightforward for the data store itself. Use replicas=1 with a placement constraint on a dedicated node, and deploy Redis Sentinel or Redis Cluster manually for high availability:

```bash
# Update service with new image (rolling update)
docker service update --image redis:7.2-alpine redis_redis

# Check rollback capability
docker service rollback redis_redis
```

## Summary

Docker Swarm simplifies Redis deployment with a single stack file that handles persistent volumes, health checks, resource limits, and rolling updates. The overlay network lets other Swarm services reach Redis by service name. For production use, pin Redis to a manager node with a persistent volume and use Swarm's built-in restart and update policies for reliability.
