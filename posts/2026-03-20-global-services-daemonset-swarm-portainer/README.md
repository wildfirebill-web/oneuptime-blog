# How to Set Up Global Services in Portainer on Swarm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Global Service, DaemonSet, Infrastructure

Description: Deploy Docker Swarm global mode services through Portainer to run exactly one container per node across the entire cluster, equivalent to Kubernetes DaemonSets.

---

A Docker Swarm global service runs exactly one replica on every node in the cluster (or every node that matches a placement constraint). This is equivalent to a Kubernetes DaemonSet. Common use cases include log collectors, metrics exporters, security agents, and network plugins.

## Global vs Replicated Services

| Mode | Behavior |
|---|---|
| `replicated` (default) | Run N replicas, scheduler decides placement |
| `global` | Run exactly one replica on every matching node |

When a new node joins the Swarm, a global service automatically deploys to it. When a node leaves, its replica is removed.

## Step 1: Deploy a Global Log Collector

In Portainer, create a stack with a global log forwarder (Filebeat example):

```yaml
version: "3.8"
services:
  log-collector:
    image: docker.elastic.co/beats/filebeat:8.12.0
    # Use global mode to run on every node
    deploy:
      mode: global
    volumes:
      # Access Docker container logs from the host
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    environment:
      - ELASTICSEARCH_HOST=elasticsearch:9200
```

## Step 2: Global Prometheus Node Exporter

A classic global service use case - run Node Exporter on every node:

```yaml
version: "3.8"
services:
  node-exporter:
    image: prom/node-exporter:latest
    deploy:
      mode: global
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)'
    ports:
      - target: 9100
        published: 9100
        mode: host    # Publish on each node's IP for Prometheus scraping
    network_mode: host
```

## Step 3: Restrict Global Service to Specific Nodes

Use placement constraints to limit the global service to a subset of nodes:

```yaml
deploy:
  mode: global
  placement:
    constraints:
      # Only run on worker nodes - skip manager nodes
      - node.role == worker
```

## Step 4: View Global Service Replicas in Portainer

In Portainer's **Services** view, global services show one replica per active node. The replica count updates automatically as nodes join or leave.

## Step 5: Update a Global Service

Updating a global service triggers a rolling update across all nodes. Use the same update_config as replicated services:

```yaml
deploy:
  mode: global
  update_config:
    parallelism: 1
    delay: 10s
    failure_action: rollback
```

## Summary

Global services are the Swarm equivalent of Kubernetes DaemonSets. They ensure infrastructure agents like log collectors, metrics exporters, and security scanners run on every node without requiring manual per-node configuration. Portainer's service view shows all global replicas and their health status in a single view.
