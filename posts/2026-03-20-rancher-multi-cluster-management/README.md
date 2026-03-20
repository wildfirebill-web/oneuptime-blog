# How to Manage Multiple Clusters from a Single Rancher Instance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Multi-Cluster, Management

Description: Learn how to effectively manage tens or hundreds of Kubernetes clusters from a single Rancher instance using cluster groups, RBAC, and Fleet-based policies.

## Introduction

One of Rancher's core strengths is its ability to manage many Kubernetes clusters from a single management plane. Whether you're managing 10 or 1,000 clusters across multiple clouds and data centers, Rancher provides the tools to organize, access, monitor, and govern them all. This guide covers the organizational patterns and operational workflows for large-scale multi-cluster management.

## Step 1: Organize Clusters with Labels

Labels are the foundation of multi-cluster organization in Rancher:

```bash
# Label clusters for organizational grouping
kubectl label cluster.management.cattle.io c-onprem-prod \
  environment=production \
  region=us-east \
  cloud=on-premises \
  tier=1

kubectl label cluster.management.cattle.io c-aws-prod-1 \
  environment=production \
  region=us-east-1 \
  cloud=aws \
  tier=1

kubectl label cluster.management.cattle.io c-gcp-dev-1 \
  environment=development \
  region=us-central1 \
  cloud=gcp \
  tier=3
```

## Step 2: Create Cluster Groups

```yaml
# cluster-group-production.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: production-clusters
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      environment: production
---
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: development-clusters
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      environment: development
---
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: us-east-clusters
  namespace: fleet-default
spec:
  selector:
    matchExpressions:
      - key: region
        operator: In
        values: [us-east, us-east-1, us-east-2]
```

```bash
kubectl apply -f cluster-group-production.yaml
kubectl get clustergroup -n fleet-default
```

## Step 3: Implement Multi-Cluster RBAC

Grant teams access to specific cluster groups:

```yaml
# rbac-platform-team.yaml
# Give the platform team cluster-admin on all production clusters
apiVersion: management.cattle.io/v3
kind: GlobalRoleBinding
metadata:
  name: platform-team-global
globalRoleName: restricted-admin
subjectName: platform-team
subjectKind: Group
```

For project-level access across clusters:

```bash
# In Rancher UI:
# Cluster → Project/Namespaces → Create Project
# → Members → Add project member with cluster selector

# Or programmatically:
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "clusterId": "c-aws-prod-1",
    "projectId": "c-aws-prod-1:p-xxxxx",
    "userId": "u-xxxxx",
    "roleTemplateId": "project-member"
  }' \
  "https://rancher.example.com/v3/projectroletemplatebindings"
```

## Step 4: Apply Policies Across All Clusters with Fleet

```yaml
# gitops/global-policies/fleet.yaml
# Apply security policies to ALL clusters
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: global-policies
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/cluster-policies
  branch: main
  paths:
    - policies/
  targets:
    - clusterSelector: {}    # Empty = all clusters
```

```yaml
# policies/network-policy-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

## Step 5: Centralized Namespace Management

```bash
# Create a project template that auto-creates namespaces
# with consistent labels across all clusters

# Create namespace via Rancher API (respects project quotas)
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "team-alpha",
    "projectId": "c-aws-prod-1:p-xxxxx",
    "annotations": {"team": "alpha", "costcenter": "engineering"}
  }' \
  "https://rancher.example.com/v3/clusters/c-aws-prod-1/namespaces"
```

## Step 6: Bulk Operations Across Clusters

```bash
#!/usr/bin/env bash
# bulk-kubectl.sh — Run kubectl commands across multiple clusters

# Get all cluster kubeconfigs from Rancher
get_kubeconfig() {
  local cluster_id="$1"
  curl -sk -X POST \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "https://rancher.example.com/v3/clusters/${cluster_id}?action=generateKubeconfig" \
    | jq -r .config
}

# Get all production cluster IDs
CLUSTER_IDS=$(curl -sk \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "https://rancher.example.com/v3/clusters?labels.environment=production" \
  | jq -r '.data[].id')

# Run a command on all production clusters
for cluster_id in ${CLUSTER_IDS}; do
  echo "=== Cluster: ${cluster_id} ==="
  kubeconfig=$(get_kubeconfig "${cluster_id}")
  echo "${kubeconfig}" > /tmp/kc-${cluster_id}
  kubectl get nodes --no-headers --kubeconfig=/tmp/kc-${cluster_id} | wc -l
  rm /tmp/kc-${cluster_id}
done
```

## Step 7: Monitor All Clusters from One Dashboard

```bash
# Install Rancher Monitoring (Prometheus + Grafana) on the management cluster
helm repo add rancher-charts https://charts.rancher.io
helm install rancher-monitoring rancher-charts/rancher-monitoring \
  -n cattle-monitoring-system \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d

# Enable cluster-level monitoring on downstream clusters via Rancher UI:
# Cluster → Monitoring → Enable
# This deploys a per-cluster Prometheus that federates to the central one
```

## Step 8: Implement Cluster Quota Policies

```yaml
# ResourceQuota applied via Fleet to all development clusters
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: default
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "0"   # No LBs in dev
```

## Conclusion

Managing multiple clusters from a single Rancher instance requires deliberate organization through labels, cluster groups, and RBAC. Fleet-based GitOps ensures consistent configuration across all clusters, while Rancher's monitoring stack provides unified observability. As your cluster count grows, automating cluster registration, labeling, and policy application becomes essential for maintaining operational efficiency at scale.
