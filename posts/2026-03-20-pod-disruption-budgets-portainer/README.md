# How to Set Up Pod Disruption Budgets via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, PDB, High Availability, Reliability, Infrastructure

Description: Configure Kubernetes Pod Disruption Budgets through Portainer to ensure minimum application availability during node drains, cluster upgrades, and maintenance operations.

---

A Pod Disruption Budget (PDB) tells Kubernetes the minimum number of pods for a workload that must remain available during voluntary disruptions (node drains, rolling upgrades, maintenance). Without PDBs, a cluster upgrade could simultaneously evict all replicas of a service and cause an outage. Portainer's manifest interface makes PDB management straightforward.

## When PDBs Are Critical

- Node maintenance and drains: `kubectl drain node-1`
- Cluster version upgrades
- Horizontal node autoscaling (scale-down)
- Manual administrator operations

## Step 1: Understand Your Availability Requirements

Before creating a PDB, decide whether to express availability as:

- `minAvailable` - minimum pods that must remain running
- `maxUnavailable` - maximum pods that can be simultaneously unavailable

## Step 2: Create a PDB via Portainer Manifest

Go to **Kubernetes > Advanced Deployment** in Portainer:

```yaml
# web-api-pdb.yaml

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-api-pdb
  namespace: production
spec:
  # Ensure at least 2 replicas are always available
  minAvailable: 2
  selector:
    matchLabels:
      app: web-api
```

Alternatively, express as a percentage:

```yaml
spec:
  # At most 20% of pods can be unavailable at any time
  maxUnavailable: "20%"
  selector:
    matchLabels:
      app: web-api
```

## Step 3: PDB for a StatefulSet

Databases and stateful services need more conservative PDBs:

```yaml
# postgres-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  # For a 3-replica Postgres cluster, only 1 can be down at a time
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
      role: replica
```

## Step 4: PDB Rules and Limits

A few important constraints:

- `minAvailable: 100%` (or equal to total replicas) will block all voluntary disruptions - drains will hang until the PDB is removed
- PDBs do not protect against hardware failures (involuntary disruptions)
- Setting `minAvailable: 0` or `maxUnavailable: 100%` effectively disables the PDB

## Step 5: Verify PDB Status in Portainer

Check PDB enforcement through the Portainer terminal or the Kubernetes namespaces view:

```bash
# Check PDB status
kubectl get pdb -n production

# Detailed view showing current and desired availability
kubectl describe pdb web-api-pdb -n production
```

The output shows:

```text
Min available:    2
Allowed disruptions: 1
Current:          3
Desired:          3
Total:            3
```

`Allowed disruptions: 1` means exactly one pod can be voluntarily evicted right now.

## Step 6: Test with a Node Drain

Simulate a maintenance event:

```bash
# Drain a node - PDB will prevent eviction if it would violate budget
kubectl drain node-worker-1 --ignore-daemonsets --delete-emptydir-data
```

If the drain would violate the PDB, it blocks and waits until the workload redistributes across other nodes.

## Summary

Pod Disruption Budgets are a lightweight but essential safety mechanism for production Kubernetes workloads. Deploying them via Portainer's manifest interface ensures they are version-controlled alongside your Deployments and StatefulSets, giving your operations team confidence that maintenance windows will not cause unexpected service disruptions.
