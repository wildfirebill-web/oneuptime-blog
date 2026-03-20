# How to Replicate Workloads Across Multiple Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Multi-Cluster, Fleet, GitOps, Workload Replication, Kubernetes, SUSE Rancher

Description: Learn how to replicate Kubernetes workloads across multiple clusters in Rancher using Fleet GitOps, including targeting specific clusters and managing per-cluster configuration overrides.

---

Replicating workloads across multiple clusters ensures high availability, geographic distribution, and disaster recovery. In Rancher, Fleet is the primary tool for deploying the same workload to multiple clusters consistently.

---

## Architecture

```text
┌─────────────────────────────────────────────────────────┐
│                    Rancher (Fleet Manager)               │
│                                                         │
│  Git Repository                                         │
│       │                                                 │
│       ├──> Fleet Bundle ──> Cluster Group A (prod-us)   │
│       │                                                 │
│       └──> Fleet Bundle ──> Cluster Group B (prod-eu)   │
└─────────────────────────────────────────────────────────┘
```

---

## Step 1: Create a GitRepo Resource Targeting Multiple Clusters

```yaml
# gitrepo-multi-cluster.yaml

apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/my-app-manifests
  branch: main
  paths:
    - manifests/

  # Target all clusters with the "env=production" label
  targets:
    - name: production-clusters
      clusterSelector:
        matchLabels:
          env: production
```

```bash
kubectl apply -f gitrepo-multi-cluster.yaml
```

---

## Step 2: Label Clusters for Targeting

In the Rancher UI or CLI, add labels to each cluster:

```bash
# Label clusters using kubectl (run against the Rancher management cluster)
kubectl label cluster.provisioning.cattle.io prod-us \
  -n fleet-default \
  env=production region=us

kubectl label cluster.provisioning.cattle.io prod-eu \
  -n fleet-default \
  env=production region=eu
```

---

## Step 3: Use Per-Cluster Overrides with fleet.yaml

Create a `fleet.yaml` in your manifests directory to apply per-cluster customizations:

```yaml
# manifests/fleet.yaml
defaultNamespace: my-app

helm:
  chart: ./chart
  releaseName: my-app

# Apply different values based on cluster labels
targetCustomizations:
  - name: us-override
    clusterSelector:
      matchLabels:
        region: us
    helm:
      values:
        replicaCount: 5
        region: us-west-2
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  topologyKey: kubernetes.io/hostname

  - name: eu-override
    clusterSelector:
      matchLabels:
        region: eu
    helm:
      values:
        replicaCount: 3
        region: eu-west-1
```

---

## Step 4: Verify Replication Status

```bash
# Check Fleet Bundle status across all clusters
kubectl get bundle -n fleet-default

# Check specific cluster deployment status
kubectl get bundledeployment -n fleet-default

# View detailed status
kubectl describe gitrepo my-app -n fleet-default
```

The output shows per-cluster readiness:

```text
Status:
  Summary:
    Ready: 2
    NotReady: 0
    DesiredReady: 2
  Clusters:
    prod-us: Ready
    prod-eu: Ready
```

---

## Step 5: Handle Configuration Differences

For configurations that differ significantly between clusters, use Kustomize overlays:

```text
manifests/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── us/
│   │   └── kustomization.yaml    # Patches for US cluster
│   └── eu/
│       └── kustomization.yaml    # Patches for EU cluster
└── fleet.yaml
```

```yaml
# fleet.yaml with Kustomize paths
targetCustomizations:
  - name: us
    clusterSelector:
      matchLabels:
        region: us
    kustomize:
      dir: overlays/us

  - name: eu
    clusterSelector:
      matchLabels:
        region: eu
    kustomize:
      dir: overlays/eu
```

---

## Step 6: Monitor Replication Drift

If a cluster's deployed state drifts from the Git source, Fleet automatically reconciles:

```bash
# Force a resync on all clusters
kubectl patch gitrepo my-app -n fleet-default \
  --type merge \
  -p '{"spec":{"forceSyncGeneration":1}}'

# Check for drift events
kubectl get events -n fleet-default \
  --field-selector reason=DriftCorrected
```

---

## Best Practices

- Use cluster labels (`region`, `env`, `tier`) rather than hardcoded cluster names in `fleet.yaml` targets - this makes it easy to add new clusters without modifying manifests.
- Keep base manifests generic and use `targetCustomizations` only for environment-specific differences like replica counts and resource limits.
- Enable drift detection by leaving Fleet's default reconciliation enabled - it will automatically revert manual changes that bypass GitOps.
