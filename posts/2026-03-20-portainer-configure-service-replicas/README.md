# How to Configure Service Replicas in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Services, Replicas, DevOps

Description: Learn how to configure replica counts, placement constraints, and scheduling modes for Docker Swarm services in Portainer.

## Introduction

Service replicas in Docker Swarm define how many instances of a container run simultaneously across the cluster. Proper replica configuration is essential for high availability, load distribution, and fault tolerance. Portainer provides an intuitive interface for configuring all aspects of service scheduling. This guide covers replica configuration in depth.

## Prerequisites

- Portainer installed on Docker Swarm
- Admin or operator access
- Understanding of basic Docker Swarm concepts

## Replica Modes Explained

### Replicated Mode (Default)

Run a fixed number of replicas distributed across available nodes:

```bash
# Run exactly 3 replicas anywhere in the cluster

docker service create --replicas 3 --name my-service nginx:alpine
```

Use replicated mode for:
- Web applications
- API services
- Databases with specific replica counts

### Global Mode

Run exactly one replica on every node in the cluster:

```bash
# Run on every node
docker service create --mode global --name my-agent myagent:latest
```

Use global mode for:
- Monitoring agents (Prometheus node exporter, Datadog agent)
- Log shippers (Filebeat, Fluentd)
- Security scanners
- The Portainer agent itself

## Configuring Replicas in Portainer

### Create a Service with Specific Replicas

1. Go to **Services → Add service**
2. In the **Scheduling** section:
   - Select **Replicated** mode
   - Set **Replicas**: `3`

### Edit Replicas on an Existing Service

1. Click the service name
2. Click **Update this service**
3. Change the **Replicas** count
4. Click **Update the service**

## Placement Constraints

Placement constraints control which nodes receive replicas:

### Available Constraint Keys

```bash
# Node properties
node.id == <node-id>
node.hostname == worker-01
node.role == manager
node.role == worker

# Node labels (set per node)
node.labels.zone == us-east-1a
node.labels.disk == ssd
node.labels.gpu == true

# Engine labels
engine.labels.operatingsystem == ubuntu
```

### Setting Node Labels

```bash
# Add a label to a node
docker node update --label-add zone=us-east-1a worker-01
docker node update --label-add disk=ssd worker-02
docker node update --label-add gpu=true gpu-node-01
```

### Configuring Constraints in Portainer

Under **Placement** when creating/editing a service:

```text
Add constraint:
  Key:      node.role
  Operator: ==
  Value:    worker
```

```text
Add constraint:
  Key:      node.labels.zone
  Operator: ==
  Value:    us-east-1a
```

### Practical Constraint Examples

```yaml
# In a stack Compose file
services:
  frontend:
    image: nginx:alpine
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.role == worker      # Only on workers, not manager
          - node.labels.zone == web  # Only on web tier nodes

  database:
    image: postgres:15
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.disk == ssd  # Only on SSD-equipped nodes
```

## Placement Preferences

In addition to constraints (hard rules), use preferences for soft distribution guidance:

```yaml
deploy:
  replicas: 6
  placement:
    preferences:
      - spread: node.labels.zone   # Spread evenly across zones
```

This distributes replicas evenly across nodes with different `zone` label values. If you have 3 zones with 6 replicas, each zone gets 2 replicas.

## Configure in Portainer Stack Editor

```yaml
# Full placement configuration example
version: "3.8"
services:
  web:
    image: nginx:alpine
    deploy:
      mode: replicated
      replicas: 6           # Desired replica count
      update_config:
        parallelism: 2      # Update 2 replicas at a time
        delay: 10s          # Wait 10s between updates
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        constraints:
          - node.role == worker
        preferences:
          - spread: node.labels.availability_zone
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
```

## Monitoring Replica Distribution

After deploying:

```bash
# See where each replica is running
docker service ps my-service --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"

# Output:
# NAME           NODE       CURRENT STATE
# my-service.1   worker-01  Running 2 minutes ago
# my-service.2   worker-02  Running 2 minutes ago
# my-service.3   worker-03  Running 2 minutes ago
```

In Portainer: click the service name to see the **Tasks** list with node assignments.

## Handling Replica Failures

Swarm automatically reschedules failed replicas:

1. A task fails or a node goes down
2. Swarm detects the departure from desired state
3. New task is scheduled on an available node

Monitor this process in Portainer's service task list or with:

```bash
# View task history including failed tasks
docker service ps my-service --no-trunc
```

## Conclusion

Properly configured service replicas are the foundation of high availability in Docker Swarm. Use replicated mode for services with specific capacity requirements and global mode for cluster-wide agents. Combine replica counts with placement constraints and preferences to achieve precise control over where your services run. Portainer's UI makes it easy to configure and adjust these settings without needing to remember CLI syntax.
