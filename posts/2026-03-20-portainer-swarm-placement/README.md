# How to Configure Swarm Service Placement Strategies in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Placement, Constraint, DevOps

Description: Configure Docker Swarm service placement strategies using node labels, affinity rules, and spread strategies via Portainer.

## Introduction

Docker Swarm's placement strategies control which nodes run which services. Portainer exposes these controls through both its UI and stack configurations. Proper placement ensures critical services run on appropriate hardware, databases avoid co-location, and stateful services have storage access.

## Placement Strategies Overview

Docker Swarm supports three placement strategies:
- **spread** (default): Distribute tasks evenly across nodes
- **binpack**: Maximize node utilization before using other nodes
- **random**: Place tasks randomly (testing only)

## Adding Node Labels via Portainer

In Portainer: **Swarm > Nodes > Select Node > Labels**

Or via CLI:
```bash
# Add labels to identify node capabilities

docker node update --label-add storage=ssd manager1
docker node update --label-add storage=hdd worker1
docker node update --label-add gpu=true worker2
docker node update --label-add region=us-east worker1
docker node update --label-add region=us-west worker2
docker node update --label-add type=database worker3

# Verify labels
docker node inspect manager1 --format '{{.Spec.Labels}}'
```

## Placement Constraints in Portainer Stacks

```yaml
# placement-demo-stack.yml
version: '3.8'

services:
  # Database: only on nodes with SSD and database label
  postgres:
    image: postgres:15
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == ssd
          - node.labels.type == database
    environment:
      POSTGRES_PASSWORD: secret

  # Cache: avoid database nodes
  redis:
    image: redis:7
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.type != database

  # Web: only on worker nodes, spread across regions
  web:
    image: nginx:latest
    deploy:
      replicas: 4
      placement:
        constraints:
          - node.role == worker
        preferences:
          - spread: node.labels.region   # Spread across regions

  # Admin tool: only on managers
  portainer-agent:
    image: portainer/agent:latest
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

  # GPU workload
  ml-inference:
    image: my-ml-app:latest
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.gpu == true
```

## Placement via Portainer UI

When deploying a service via Portainer:
1. Go to **Swarm > Services > Add Service**
2. In **Placement** tab:
   - Add **Constraints**: `node.labels.type == database`
   - Add **Preferences**: `spread: node.labels.region`

## Dynamic Placement with Node Updates

```bash
# Temporarily move services by updating node labels
# Before maintenance: remove the database label from a node
docker node update --label-rm type manager1

# Services with constraint "type == database" will migrate away
# Verify migration
docker service ps myapp_postgres

# After maintenance: restore label
docker node update --label-add type=database manager1
```

## Common Placement Patterns

```yaml
# Pattern 1: HA database with replica on different physical hosts
db-primary:
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.labels.rack == rack-a

db-replica:
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.labels.rack == rack-b  # Different rack for true HA

# Pattern 2: Stateful service on storage node
stateful-app:
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.labels.has-nfs-mount == true

# Pattern 3: Spread across availability zones
web-tier:
  deploy:
    replicas: 3
    placement:
      preferences:
        - spread: node.labels.availability-zone
```

## Conclusion

Proper placement strategies in Docker Swarm ensure services run on appropriate hardware, maintain high availability across failure domains, and prevent resource contention. Portainer makes managing these strategies accessible through both its stack editor and service configuration UI, allowing operators to define complex placement rules without memorizing Docker CLI flags.
