# How to Manage Multiple Clusters from a Single Rancher Instance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Multi-Cluster, Kubernetes, Cluster Management, DevOps, RBAC, Federation

Description: Learn how to import, provision, and manage multiple Kubernetes clusters from a single Rancher instance, covering cluster registration, RBAC, and centralized operations.

---

One of Rancher's core strengths is its ability to manage dozens or hundreds of Kubernetes clusters from a single control plane. This guide covers the key workflows for multi-cluster management.

---

## Cluster Import vs. Provisioning

Rancher supports two modes:

| Method | Use Case |
|---|---|
| **Import** | Existing clusters (EKS, GKE, AKS, on-prem) |
| **Provision** | Rancher creates and manages the cluster lifecycle |

---

## Step 1: Import an Existing Cluster

In Rancher UI, go to **Cluster Management > Import Existing**. Choose the cluster type, give it a name, then run the provided command on the cluster:

```bash
# Rancher generates a unique import manifest per cluster

# Run this on the target cluster to register it
kubectl apply -f https://rancher.example.com/v3/import/<token>.yaml

# Verify the cattle-cluster-agent is running
kubectl get pods -n cattle-system
```

The cluster appears in Rancher UI once the agent connects.

---

## Step 2: Provision a New RKE2 Cluster

Use Rancher to provision a new RKE2 cluster on existing Linux nodes:

```yaml
# cluster-rke2.yaml (applied via Rancher API or Fleet)
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production-us-east
  namespace: fleet-default
spec:
  rkeConfig:
    machinePools:
      - name: control-plane
        quantity: 3
        machineConfigRef:
          kind: Amazonec2Config
          name: rke2-control-plane
        controlPlaneRole: true
        etcdRole: true
      - name: workers
        quantity: 5
        machineConfigRef:
          kind: Amazonec2Config
          name: rke2-worker
        workerRole: true
```

---

## Step 3: Organize Clusters with Labels and Annotations

Labels help with targeting in Fleet and filtering in the UI:

```bash
# Label clusters for environment and region
kubectl annotate cluster.management.cattle.io/c-xxxxx \
  field.cattle.io/description="Production US East" \
  -n fleet-default

kubectl label cluster.management.cattle.io/c-xxxxx \
  env=production \
  region=us-east \
  tier=frontend \
  -n fleet-default
```

---

## Step 4: Configure Cluster-Level RBAC

Assign users to clusters using Rancher's ClusterRoleTemplateBindings:

```yaml
# Bind a user to the cluster-member role on a specific cluster
apiVersion: management.cattle.io/v3
kind: ClusterRoleTemplateBinding
metadata:
  name: jane-cluster-member
  namespace: c-xxxxx
roleTemplateName: cluster-member
userPrincipalName: local://u-jane
```

Rancher's built-in roles include:
- `cluster-owner` - full control
- `cluster-member` - read + deploy access
- `read-only` - view-only

---

## Step 5: Use kubectl for Multiple Clusters

Download per-cluster kubeconfigs from the Rancher UI, or use the Rancher CLI:

```bash
# Install rancher CLI
curl -Lo rancher https://github.com/rancher/cli/releases/download/v2.8.0/rancher-linux-amd64-v2.8.0.tar.gz

# Login
rancher login https://rancher.example.com --token <api-token>

# List clusters
rancher cluster ls

# Switch context to a specific cluster
rancher context switch

# Run kubectl against a specific cluster
rancher kubectl --cluster production-us-east get nodes
```

---

## Step 6: Centralized Alerting Across Clusters

In Rancher UI, configure a **global** alerting receiver at **Cluster Management > Advanced > Global Alerting** to receive alerts from all clusters in one place. Pair this with Rancher's built-in Prometheus and Alertmanager or an external observability platform.

---

## Best Practices

- Use **cluster groups** in Fleet to apply policies to logical collections of clusters.
- Keep the Rancher management cluster separate from workload clusters for stability.
- Enable Rancher's **CIS scan** on all clusters to baseline security posture across environments.
- Store cluster provisioning configs in Git and apply with Fleet for infrastructure-as-code.
