# How to View Swarm Cluster Details in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Swarm, Monitoring, DevOps

Description: Learn how to view and interpret Docker Swarm cluster details, node information, and cluster health in Portainer.

## Introduction

Portainer provides a comprehensive view of your Docker Swarm cluster, including node status, resource utilization, and cluster-wide service distribution. Understanding how to navigate these views helps you monitor cluster health and quickly identify issues. This guide covers all the ways to inspect your Swarm cluster through Portainer.

## Prerequisites

- Portainer installed on Docker Swarm
- At least one manager and one worker node
- Admin access to Portainer

## Step 1: Access the Swarm Environment

1. Log in to Portainer
2. On the **Home** screen, click on your Swarm environment
3. The Swarm environment dashboard loads

## Step 2: View the Dashboard Overview

The Swarm dashboard shows a summary:

```
Docker Swarm Environment
─────────────────────────────
Services:          12         Running Swarm services
Configs:            3         Docker configs stored
Secrets:            8         Docker secrets
Networks:           7         Overlay networks
Volumes:           24         Named volumes
Nodes:              5         Active cluster nodes
```

## Step 3: View Swarm Node Details

Navigate to **Swarm** in the sidebar to see node details:

### Cluster Information

```
Swarm ID:    9dh8x4...7f2a
Managers:    1
Workers:     4
Total Nodes: 5
```

### Node Table

Each node shows:

| Column | Description |
|--------|-------------|
| ID | Node short ID |
| Hostname | Node hostname |
| Role | Manager or Worker |
| Status | Active, Down, or Paused |
| Availability | Active, Pause, or Drain |
| Manager Status | Leader, Reachable, or Unreachable |
| Engine Version | Docker engine version |

## Step 4: Inspect a Specific Node

Click on a node row to see detailed information:

```
Node ID:           abc123def456
Hostname:          worker-01
IP Address:        10.0.1.11
Role:              Worker
Status:            Ready
Availability:      Active
OS:                linux
Architecture:      x86_64
CPU Count:         4
Memory:            7.78 GiB
Engine Version:    24.0.7
Labels:
  - zone=us-east-1a
  - ssd=true
```

## Step 5: View Running Tasks per Node

In the node detail view, scroll down to see tasks (containers) running on that node:

```
Tasks on worker-01
──────────────────────────────────────────────────────
ID           Service              State     Image
7d8f93      web_frontend.3       Running   nginx:alpine
a1b2c3      api_backend.1        Running   myapi:v2.1
f9e8d7      portainer_agent.3    Running   portainer/agent:latest
```

This shows which service replicas are scheduled on each node.

## Step 6: View Swarm Visualizer

Portainer includes a visual representation of your Swarm cluster (see the Swarm Visualizer guide for details). To access it:

1. Navigate to **Swarm → Visualizer**
2. The grid view shows nodes as columns and tasks as colored boxes

## Step 7: Monitor Node Resource Usage

To view CPU and memory usage per node:

1. Go to **Nodes** list
2. Click on a node
3. Click **Stats** in the node detail view

For aggregate cluster statistics, use the Portainer dashboard or deploy a monitoring stack (Prometheus + Grafana) to your Swarm.

## Step 8: Check Cluster Network Information

View overlay networks used for inter-service communication:

1. Navigate to **Networks** in the sidebar
2. Look for networks with **Overlay** driver
3. Click a network to see:
   - Connected services
   - IP address ranges (IPAM)
   - Network ID

```bash
# View overlay networks from CLI
docker network ls --filter driver=overlay

# Inspect a specific overlay network
docker network inspect ingress
```

## Step 9: View Swarm Configs and Secrets

Docker Swarm configs and secrets are cluster-wide:

- **Configs** → Navigate to **Swarm → Configs**
- **Secrets** → Navigate to **Swarm → Secrets**

These are distinct from environment variables — configs and secrets are distributed to containers that reference them in service definitions.

## Interpreting Node Status

| Status | Availability | Meaning |
|--------|-------------|---------|
| Ready | Active | Node is healthy and accepts tasks |
| Ready | Pause | Node is healthy but won't receive new tasks |
| Ready | Drain | Node is draining; existing tasks are rescheduled |
| Down | - | Node is unreachable |
| Disconnected | - | Node lost contact with manager |

## Troubleshooting Unhealthy Nodes

```bash
# Check why a node is down
docker node inspect worker-01 --pretty

# Force remove a permanently down node
docker node rm worker-01

# Update a node's availability
docker node update --availability pause worker-01
docker node update --availability active worker-01
```

## Conclusion

Portainer's Swarm cluster views give you complete visibility into your multi-node Docker infrastructure. Regularly review node status, task distribution, and resource utilization to proactively identify and resolve issues before they impact your applications. Use the visualizer for a quick cluster overview and the node detail views for in-depth troubleshooting.
