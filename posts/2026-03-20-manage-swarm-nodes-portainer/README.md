# How to Manage Swarm Nodes in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Docker Swarm, Portainer, Container Management, DevOps

Description: Learn how to view, inspect, and manage Docker Swarm nodes directly from the Portainer UI.

## Overview

Docker Swarm turns a group of Docker hosts into a single virtual Docker engine. Portainer gives you a graphical interface to manage all nodes in your Swarm cluster without touching the command line.

## Viewing Swarm Nodes

After connecting Portainer to a Docker Swarm endpoint, navigate to **Swarm > Nodes** in the left sidebar. You will see a list of all manager and worker nodes with their:

- **Hostname** and **IP address**
- **Status** (Ready/Down)
- **Availability** (Active/Pause/Drain)
- **Role** (Manager/Worker)
- **Engine version**

## Inspecting a Node

Click on any node name to open the detail view. Here you can see the full node spec including labels, resources, and platform information.

The following CLI command shows the equivalent information via the Docker CLI:

```bash
# List all swarm nodes with their status and availability
docker node ls

# Inspect a specific node by its ID or hostname
docker node inspect <node-id> --pretty
```

## Changing Node Availability

You can drain a node before maintenance directly from Portainer by clicking the node and selecting **Drain** from the Availability dropdown. This moves all running tasks off the node.

```bash
# Equivalent CLI command to drain a node
docker node update --availability drain <node-id>

# Reactivate a node after maintenance
docker node update --availability active <node-id>
```

## Promoting and Demoting Nodes

To change a worker to a manager (or vice versa), click the node in Portainer and use the **Role** dropdown to promote or demote it.

```bash
# Promote a worker to manager via CLI
docker node promote <node-id>

# Demote a manager to worker via CLI
docker node demote <node-id>
```

## Adding Labels to Nodes

Node labels are used for placement constraints. In Portainer, open a node and scroll to the **Labels** section to add key-value pairs.

```bash
# Add a label to a node via CLI
docker node update --label-add env=production <node-id>

# Remove a label from a node
docker node update --label-rm env <node-id>
```

## Removing a Node from the Swarm

To remove a node, first drain it, then remove it from the Portainer UI by clicking **Remove** on the node detail page.

```bash
# Remove a node from the swarm via CLI (must be drained first)
docker node rm <node-id>
```

## Conclusion

Portainer makes Swarm node management accessible without requiring deep knowledge of Docker CLI commands. Use it to monitor node health, drain nodes for maintenance, manage roles, and apply labels for workload placement.
