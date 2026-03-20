# How to Manage Cluster Groups in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Groups, Fleet, Management

Description: Organize and manage clusters at scale using Rancher Fleet Cluster Groups to apply policies, deploy workloads, and monitor groups of related clusters.

## Introduction

As the number of Rancher-managed clusters grows, managing them individually becomes impractical. Cluster Groups in Rancher Fleet let you define logical sets of clusters and target GitOps deployments, policies, and monitoring configurations at the group level rather than individual clusters. This guide covers creating, managing, and leveraging Cluster Groups effectively.

## What are Cluster Groups?

A `ClusterGroup` is a Fleet resource that dynamically selects clusters based on label selectors. When you target a deployment at a ClusterGroup, Fleet automatically deploys to all current and future clusters that match the selector — no manual cluster enumeration required.

## Step 1: Label Your Clusters

Labels are the foundation for all ClusterGroup selectors:

```bash
# Apply consistent labels to all clusters as you register them

# Production clusters in AWS
kubectl label cluster.management.cattle.io c-aws-prod-1 \
  environment=production cloud=aws region=us-east-1 tier=1

kubectl label cluster.management.cattle.io c-aws-prod-2 \
  environment=production cloud=aws region=eu-west-1 tier=1

# Development clusters
kubectl label cluster.management.cattle.io c-gcp-dev-1 \
  environment=development cloud=gcp region=us-central1 tier=3

# Edge clusters
kubectl label cluster.management.cattle.io c-edge-retail-1 \
  environment=production cloud=on-premises type=edge location=store-001

kubectl label cluster.management.cattle.io c-edge-retail-2 \
  environment=production cloud=on-premises type=edge location=store-002
```

## Step 2: Create Cluster Groups

```yaml
# cluster-groups.yaml
---
# All production clusters
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: production
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      environment: production
---
# All AWS clusters
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: aws-clusters
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      cloud: aws
---
# Edge clusters (using expression selector)
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: edge-clusters
  namespace: fleet-default
spec:
  selector:
    matchExpressions:
      - key: type
        operator: In
        values: [edge]
      - key: environment
        operator: In
        values: [production, staging]
---
# Tier 1 (critical) clusters
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: tier1-critical
  namespace: fleet-default
spec:
  selector:
    matchLabels:
      tier: "1"
```

```bash
kubectl apply -f cluster-groups.yaml
kubectl get clustergroup -n fleet-default
```

## Step 3: Deploy to Cluster Groups

Target GitRepo deployments at a ClusterGroup:

```yaml
# gitrepo-security-policies.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: security-policies
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/security-policies
  branch: main
  targets:
    # Apply base policies to ALL clusters
    - name: all-clusters
      clusterSelector: {}

    # Apply stricter policies to production clusters
    - name: production-hardening
      clusterGroup: production          # ← Reference ClusterGroup by name
      helm:
        values:
          strictMode: true
          auditLevel: RequestResponse
---
# gitrepo-monitoring.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: monitoring-stack
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/monitoring
  branch: main
  targets:
    - name: production-monitoring
      clusterGroup: tier1-critical
      helm:
        values:
          retention: 30d
          storage: 100Gi

    - name: edge-monitoring
      clusterGroup: edge-clusters
      helm:
        values:
          retention: 3d
          storage: 10Gi    # Smaller storage for edge nodes
```

## Step 4: Check ClusterGroup Status

```bash
# View group membership
kubectl get clustergroup -n fleet-default -o json \
  | jq '.items[] | {name: .metadata.name, clusterCount: .status.clusterCount}'

# Detailed group status
kubectl describe clustergroup -n fleet-default production

# List which clusters are in a group
kubectl get cluster -n fleet-default \
  -l environment=production \
  -o custom-columns="NAME:.metadata.name,REGION:.metadata.labels.region,CLOUD:.metadata.labels.cloud"
```

## Step 5: Apply RBAC at the Cluster Group Level

```bash
# Grant a team read-only access to all production clusters
# using Rancher's ClusterRoleTemplateBinding

# In Rancher UI: Global → Users & Authentication → Roles
# Create a GlobalRole and assign it to a group

# Via API:
curl -sk -X POST \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "globalRoleName": "read-only",
    "userId": null,
    "groupPrincipalId": "keycloak_group://prod-readonly-team"
  }' \
  "https://rancher.example.com/v3/globalrolebindings"
```

## Step 6: Monitor Cluster Group Health

```bash
# Check BundleDeployment status for the production ClusterGroup
kubectl get bundledeployment -A \
  | grep production \
  | awk '{print $1, $2, $4, $5}'

# Get a summary of ready vs not-ready clusters per group
kubectl get clustergroup -n fleet-default -o json | jq -r '
  .items[] |
  "\(.metadata.name): \(.status.readyClusters // 0)/\(.status.clusterCount // 0) clusters ready"
'
```

## Step 7: Automate Group-Level Operations

```bash
#!/usr/bin/env bash
# drain-group.sh — Cordon all nodes in a cluster group for maintenance

GROUP_NAME="${1:?Usage: drain-group.sh <group-name>}"

# Get all clusters in the group
CLUSTERS=$(kubectl get cluster -n fleet-default \
  -l "$(kubectl get clustergroup -n fleet-default ${GROUP_NAME} \
    -o jsonpath='{.spec.selector.matchLabels}' | jq -r 'to_entries|map("\(.key)=\(.value)")|join(",")')" \
  -o jsonpath='{.items[*].metadata.name}')

for cluster in ${CLUSTERS}; do
  echo "Processing cluster: ${cluster}"
  kubeconfig=$(get_cluster_kubeconfig "${cluster}")

  # Cordon all nodes
  kubectl --kubeconfig="${kubeconfig}" \
    cordon $(kubectl --kubeconfig="${kubeconfig}" get nodes -o name)

  echo "All nodes in ${cluster} cordoned"
done
```

## Conclusion

Cluster Groups transform Rancher from a per-cluster management tool into a true fleet management platform. By defining groups based on environment, cloud provider, tier, or any other attribute, and targeting GitRepo deployments and policies at these groups, you achieve consistent configuration across all members without touching each cluster individually. As your cluster estate evolves, newly registered clusters automatically inherit the correct policies and workloads as soon as their labels match a ClusterGroup selector.
