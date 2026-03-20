# How to Configure Fleet Bundle Targets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Fleet, GitOps, Rancher, Kubernetes, Bundles

Description: Learn how to configure Fleet bundle targets to control which clusters receive specific application deployments using selectors, cluster names, and cluster groups.

## Introduction

Fleet bundle targets define the rules for which Kubernetes clusters should receive a particular bundle of resources. Configuring targets correctly is fundamental to managing multi-cluster deployments - it allows you to specify exactly where your applications should run, whether by individual cluster name, labels, or cluster groups.

Targets can be configured both in the `fleet.yaml` file within your Git repository and in the `GitRepo` resource itself.

## Prerequisites

- Fleet installed and operational
- Multiple clusters registered in Fleet
- `kubectl` access to the Fleet manager
- Basic understanding of Kubernetes label selectors

## Understanding Fleet Bundle Targeting

Fleet evaluates targets in order and applies the **first matching target** to each cluster. This precedence model allows you to define general rules and override them for specific clusters.

Target matching methods:
1. **clusterName**: Exact match on a specific cluster
2. **clusterSelector**: Label-based matching
3. **clusterGroup**: Reference to a named cluster group
4. **clusterGroupSelector**: Label-based group matching

## Basic Target Configuration in fleet.yaml

```yaml
# fleet.yaml - Bundle target configuration

namespace: my-app

targets:
  # Target a specific cluster by name
  - name: local-deployment
    clusterName: local

  # Target clusters matching specific labels
  - name: production-clusters
    clusterSelector:
      matchLabels:
        env: production
        region: us-west-2

  # Target a predefined cluster group
  - name: staging-group
    clusterGroup: staging-clusters

  # Default target - match all remaining clusters
  - name: default
    clusterSelector: {}
```

## Configuring Targets with Customizations Per Target

Each target can include custom Helm values, Kustomize patches, or namespace overrides:

```yaml
# fleet.yaml - Per-target customizations
namespace: my-app

targets:
  # Staging: reduced resources, verbose logging
  - name: staging
    clusterSelector:
      matchLabels:
        env: staging
    helm:
      values:
        replicaCount: 1
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
        logging:
          level: debug

  # Production: full resources, minimal logging
  - name: production
    clusterSelector:
      matchLabels:
        env: production
    helm:
      values:
        replicaCount: 3
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
        logging:
          level: error
```

## Using matchExpressions in clusterSelector

For more complex label matching:

```yaml
# fleet.yaml - Advanced label expressions
targets:
  - name: large-clusters
    clusterSelector:
      matchLabels:
        tier: large
      matchExpressions:
        # Target clusters in either us-east or us-west regions
        - key: region
          operator: In
          values:
            - us-east-1
            - us-west-2
        # Exclude clusters that are under maintenance
        - key: maintenance
          operator: DoesNotExist
```

## Configuring Targets in GitRepo Resources

You can also set default targets at the GitRepo level:

```yaml
# gitrepo-with-targets.yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/my-app
  branch: main

  # GitRepo-level targets apply to all bundles
  # unless overridden in fleet.yaml
  targets:
    - name: dev-clusters
      clusterSelector:
        matchLabels:
          env: dev

    - name: prod-clusters
      clusterSelector:
        matchLabels:
          env: production
```

## Restricting Deployments with Namespace Isolation

Use `targetNamespace` to constrain where resources are deployed:

```yaml
# fleet.yaml - Namespace-restricted targets
targets:
  - name: team-a-clusters
    clusterSelector:
      matchLabels:
        team: team-a
    # All resources go into the team-a namespace
    targetNamespace: team-a-apps

  - name: team-b-clusters
    clusterSelector:
      matchLabels:
        team: team-b
    targetNamespace: team-b-apps
```

## Labeling Clusters for Targeting

To use label-based targeting, you first need to label your clusters:

```bash
# Label a cluster via kubectl (Fleet cluster resource)
kubectl label cluster my-cluster-name \
  env=production \
  region=us-west-2 \
  team=platform \
  -n fleet-default

# Verify the labels
kubectl get cluster my-cluster-name -n fleet-default --show-labels
```

Via the Rancher UI:
1. Navigate to **Continuous Delivery > Clusters**
2. Click on the cluster you want to label
3. Edit the cluster and add labels

## Testing Target Matching

After configuring targets, verify which clusters are matched:

```bash
# Check which bundles are deployed where
kubectl get bundledeployments -A

# Get detailed bundle deployment status
kubectl get bundledeployment -A -o wide

# Check a specific bundle's targets
kubectl describe bundle my-app -n fleet-default | grep -A 20 "Targets:"
```

## Handling Target Conflicts

When a cluster matches multiple targets, Fleet uses the first match. Order targets from most specific to most general:

```yaml
# fleet.yaml - Correct target ordering (specific first)
targets:
  # Most specific: production clusters in EU
  - name: prod-eu
    clusterSelector:
      matchLabels:
        env: production
        region: eu-central-1

  # Less specific: all production clusters
  - name: prod-all
    clusterSelector:
      matchLabels:
        env: production

  # Least specific: catch-all for everything else
  - name: default
    clusterSelector: {}
```

## Conclusion

Fleet bundle targets give you precise control over application deployment topology across your cluster fleet. By combining cluster name targeting, label selectors, and cluster groups, you can implement sophisticated deployment strategies - from simple all-cluster deployments to complex per-region, per-environment configurations. Careful target ordering and labeling strategy are key to maintaining a clean and predictable multi-cluster delivery pipeline.
