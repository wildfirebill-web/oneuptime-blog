# How to Use SUSE Observability for Kubernetes Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SUSE Observability, Kubernetes, Monitoring, Topology, Health, Metric, SUSE Rancher

Description: Learn how to use SUSE Observability to monitor Kubernetes cluster health, navigate topology maps, investigate incidents, and set up health monitors for workloads.

---

SUSE Observability provides a topology-aware view of your Kubernetes cluster - showing not just metrics, but the relationships between components and how changes propagate through your system.

---

## Key Concepts

| Concept | Description |
|---|---|
| Component | Any Kubernetes resource (Pod, Deployment, Service, Node) |
| Relation | A dependency or connection between components |
| Health state | GREEN, ORANGE, RED, or UNKNOWN for each component |
| Perspective | A filtered view of the topology |
| Monitor | A rule that evaluates component health over time |

---

## Navigating the Topology View

After logging in to SUSE Observability:

1. Go to **Views** → **Kubernetes**
2. Select your cluster from the left panel
3. The topology map shows all components and their connections
4. Click any component to see its health, metrics, events, and related components

---

## Step 1: Explore Cluster Health

```ini
Topology Map Navigation:
  ┌────────────────────────────────────────┐
  │  Filter bar: namespace, label, type    │
  ├────────────────────────────────────────┤
  │                                        │
  │   [Node] ──> [Pod] ──> [Service]       │
  │              │                         │
  │              └──> [ConfigMap]          │
  │                                        │
  │   Health: ● GREEN  ● ORANGE  ● RED     │
  └────────────────────────────────────────┘
```

Filter the topology by namespace to focus on a specific workload:

```text
Views → Kubernetes → Filter by: namespace = production
```

---

## Step 2: Investigate a Failing Component

When a component shows RED health:

1. Click the component in the topology map
2. In the right panel, click **Health** to see active health violations
3. Click **Metrics** to view CPU, memory, and network graphs
4. Click **Events** to see Kubernetes events for this component
5. Click **Related** to see which other components are affected

---

## Step 3: Use the CLI for Topology Queries

SUSE Observability provides a query language (STQL) for searching topology:

```stql
# Find all pods in the production namespace with RED health

type = "kubernetes-pod"
  AND label = "namespace:production"
  AND healthState = "RED"

# Find all deployments with more than 0 failed replicas
type = "kubernetes-deployment"
  AND metrics.kube_deployment_status_replicas_unavailable > 0

# Find components changed in the last hour
type IN ("kubernetes-deployment", "kubernetes-statefulset")
  AND lastUpdated > "-1h"
```

Enter STQL queries in the search bar of the topology view.

---

## Step 4: Set Up Health Monitors

Health monitors alert you when a component's state changes:

```yaml
# SUSE Observability supports monitors defined via the API
# Example: Alert when pod restart count exceeds threshold
POST /api/v1/monitors
{
  "name": "High Pod Restart Count",
  "query": {
    "type": "MetricQuery",
    "metric": "kube_pod_container_status_restarts_total",
    "threshold": {
      "critical": 10,
      "warning": 5
    },
    "window": "5m"
  },
  "componentFilter": "type = \"kubernetes-pod\""
}
```

---

## Step 5: Monitor Node Health

```bash
# From the UI, navigate to:
# Views → Kubernetes → Nodes

# Each node shows:
# - CPU and memory utilization
# - Running pods
# - Conditions (Ready, MemoryPressure, DiskPressure)
# - Recent events

# To check node-level health from the CLI:
kubectl get nodes -o wide

# Cross-reference with Observability by filtering:
# type = "kubernetes-node" AND healthState = "ORANGE"
```

---

## Step 6: Track Changes Over Time

SUSE Observability records every change to your topology:

1. Click any component in the topology map
2. Select the **Changes** tab in the right panel
3. View a timeline of configuration changes, restarts, and scaling events
4. Use the timeline slider to see the topology state at any point in time

This makes it easy to correlate a performance degradation with a specific deployment or configuration change.

---

## Step 7: Create a Custom View

Save a filtered topology view for your team:

```text
1. Apply filters: namespace = production, type = Pod
2. Click "Save View" in the top right
3. Name the view: "Production Pods"
4. Set visibility: Team or Personal
5. Share the view URL with your team
```

---

## Useful Metric Queries

From the component metrics panel, use these metric names:

```text
# CPU usage by pod
container_cpu_usage_seconds_total

# Memory usage by pod
container_memory_working_set_bytes

# Pod restart count
kube_pod_container_status_restarts_total

# Number of unavailable replicas
kube_deployment_status_replicas_unavailable

# Node disk pressure
kube_node_status_condition{condition="DiskPressure",status="true"}
```

---

## Best Practices

- Use topology filtering (by namespace or label) to create focused views for each team rather than giving everyone access to the full cluster topology.
- Set up health monitors for critical workloads so your team is alerted before users notice issues.
- Use the Change History feature to correlate incidents with recent deployments - this dramatically reduces mean time to identify the root cause.
