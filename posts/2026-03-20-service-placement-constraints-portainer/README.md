# How to Configure Service Placement Constraints in Portainer on Swarm (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker Swarm, Portainer, Placement Constraints, Container Orchestration, DevOps

Description: Learn how to configure Docker Swarm service placement constraints in Portainer to control which nodes run specific services.

## What Are Placement Constraints?

Placement constraints restrict which Swarm nodes a service's tasks can run on. Common use cases include pinning services to specific hardware (e.g., GPU nodes), environments (production vs. staging), or roles (manager vs. worker).

## Setting Constraints in Portainer

When creating or editing a Swarm service in Portainer:

1. Go to **Services** and click **Add service** (or edit an existing one).
2. Scroll to the **Placement** section.
3. Click **Add constraint** and enter the constraint expression.

## Constraint Syntax

Constraints use the format `node.<attribute> == <value>` or `node.labels.<key> == <value>`.

```bash
# Constrain to manager nodes only

node.role == manager

# Constrain to a specific hostname
node.hostname == worker-01

# Constrain using a custom label (must be set on the node first)
node.labels.env == production

# Exclude a node by hostname
node.hostname != worker-02
```

## Adding Node Labels for Constraints

First, label your nodes so services can target them:

```bash
# Add a label to a node
docker node update --label-add env=production <node-id>

# Verify the label was added
docker node inspect <node-id> --pretty | grep Labels -A5
```

## Using Constraints in a Compose File

```yaml
version: "3.8"

services:
  database:
    image: postgres:15
    deploy:
      replicas: 1
      placement:
        constraints:
          # Only run on nodes labeled as production
          - node.labels.env == production
          # Only run on worker nodes (not managers)
          - node.role == worker
```

## Combining Multiple Constraints

Multiple constraints are evaluated as AND conditions - all must be satisfied for a task to be placed on a node:

```yaml
deploy:
  placement:
    constraints:
      - node.labels.region == us-east
      - node.labels.disk == ssd
      - node.role == worker
```

## Verifying Constraint Application

```bash
# Check where service tasks are running
docker service ps my-service

# If tasks are in "pending" state, constraints may be too restrictive
docker service inspect my-service --pretty
```

## Conclusion

Placement constraints give you fine-grained control over workload distribution in Docker Swarm. Portainer makes it easy to add and modify these constraints without writing CLI commands directly.
