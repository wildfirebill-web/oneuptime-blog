# How to View and Filter ConfigMaps in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, ConfigMap, Management, DevOps

Description: Learn how to view, search, and filter ConfigMaps in Portainer to efficiently manage application configuration across your Kubernetes cluster.

## Introduction

As Kubernetes clusters grow, the number of ConfigMaps can become large and unwieldy. A single namespace may have dozens of ConfigMaps for different applications, middleware components, and feature configurations. Portainer provides a centralized view for browsing and filtering ConfigMaps, while kubectl offers powerful filtering options for automation and scripting. This guide covers efficiently finding and managing ConfigMaps in Portainer.

## Prerequisites

- Portainer with Kubernetes environment
- ConfigMaps deployed in one or more namespaces

## Step 1: Navigate to ConfigMaps in Portainer

1. Select your Kubernetes environment in Portainer
2. Select a namespace from the dropdown (or "All namespaces")
3. In the sidebar, click **ConfigMaps & Secrets**
4. Select the **ConfigMaps** tab

The list displays:
```text
Name                 Namespace      Keys    Created
app-config           production     8       2 days ago
db-config            production     5       2 days ago
nginx-config         production     2       1 day ago
feature-flags        production     12      3 hours ago
monitoring-config    monitoring     6       5 days ago
```

## Step 2: Search and Filter ConfigMaps

In the Portainer ConfigMaps list:

1. Use the **search box** at the top of the list to filter by name
   - Type `nginx` to show only ConfigMaps with "nginx" in their name
   - Type `config` to show all ConfigMaps containing "config"

2. Use the **namespace filter** to narrow to a specific namespace

3. Sort by clicking column headers:
   - Sort by **Name** for alphabetical browsing
   - Sort by **Created** to find recently modified configs

## Step 3: View ConfigMap Details in Portainer

Click on a ConfigMap name to view its details:

1. **Metadata section**: name, namespace, labels, annotations, creation time
2. **Data section**: all key-value pairs with their values
3. **Events section**: recent events related to this ConfigMap

For large multi-line values (like config files), the detail view shows the full content.

## Step 4: Filter ConfigMaps via kubectl

Command-line filtering provides more powerful options:

```bash
# List all ConfigMaps in a namespace

kubectl get configmaps -n production

# List with label selector
kubectl get configmaps -n production -l app=my-app

# List across all namespaces
kubectl get configmaps --all-namespaces

# Filter by name pattern (using grep)
kubectl get configmaps -n production | grep nginx

# Show ConfigMaps with specific annotation
kubectl get configmaps -n production \
  -o json | jq -r '.items[] |
  select(.metadata.annotations["managed-by"] == "helm") |
  .metadata.name'

# List ConfigMaps sorted by creation time
kubectl get configmaps -n production \
  --sort-by='.metadata.creationTimestamp'

# Show only names
kubectl get configmaps -n production -o name
```

## Step 5: View ConfigMap Contents

```bash
# View a specific ConfigMap
kubectl describe configmap app-config -n production

# Get all data as JSON
kubectl get configmap app-config -n production -o json | jq '.data'

# Get a specific key value
kubectl get configmap app-config -n production \
  -o jsonpath='{.data.DATABASE_HOST}'

# Get all keys (not values)
kubectl get configmap app-config -n production \
  -o jsonpath='{.data}' | \
  python3 -c "import sys, json; print('\n'.join(json.load(sys.stdin).keys()))"

# View large config file values
kubectl get configmap nginx-config -n production \
  -o jsonpath='{.data.nginx\.conf}'
```

## Step 6: Find ConfigMaps Referenced by Pods

Identify which pods use a specific ConfigMap:

```bash
# Find pods that mount a ConfigMap as a volume
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(
    .spec.volumes[]?.configMap.name == "app-config"
  ) | .metadata.name'

# Find pods that reference a ConfigMap via envFrom
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(
    .spec.containers[].envFrom[]?.configMapRef.name == "app-config"
  ) | .metadata.name'

# Find deployments using a ConfigMap
kubectl get deployments -n production -o json | \
  jq -r '.items[] | select(
    (.spec.template.spec.volumes[]?.configMap.name == "app-config") or
    (.spec.template.spec.containers[].envFrom[]?.configMapRef.name == "app-config")
  ) | .metadata.name'
```

## Step 7: Identify Unused ConfigMaps

Find ConfigMaps that no pods reference (candidates for cleanup):

```bash
#!/bin/bash
# Find potentially unused ConfigMaps
NAMESPACE=production

echo "ConfigMaps in namespace $NAMESPACE:"
kubectl get configmap -n $NAMESPACE -o name | sed 's/configmap\///'

echo ""
echo "ConfigMaps referenced by pods:"
kubectl get pods -n $NAMESPACE -o json | \
  jq -r '[.items[].spec |
    (.volumes[]?.configMap.name // empty),
    (.containers[].envFrom[]?.configMapRef.name // empty)
  ] | unique | .[]' | sort -u

# Manually compare the two lists to find unreferenced ConfigMaps
```

## Step 8: Export ConfigMaps for Backup or Migration

```bash
# Export all ConfigMaps from a namespace (excluding system ones)
kubectl get configmaps -n production \
  -o yaml > production-configmaps-backup.yaml

# Export specific ConfigMap
kubectl get configmap app-config -n production \
  -o yaml > app-config-backup.yaml

# Export without cluster-specific fields (for portability)
kubectl get configmap app-config -n production \
  -o yaml | \
  grep -v "resourceVersion\|uid\|selfLink\|creationTimestamp: null" \
  > app-config-clean.yaml

# Export all ConfigMaps to individual files
for cm in $(kubectl get configmaps -n production -o name); do
  name=$(echo $cm | sed 's/configmap\///')
  kubectl get configmap $name -n production -o yaml > "${name}.yaml"
done
```

## Step 9: Compare ConfigMaps Across Namespaces

Verify staging and production configs match (except environment-specific values):

```bash
# Get ConfigMap keys from production
kubectl get configmap app-config -n production \
  -o jsonpath='{.data}' | python3 -m json.tool | \
  python3 -c "import sys, json; print(sorted(json.load(sys.stdin).keys()))"

# Get ConfigMap keys from staging
kubectl get configmap app-config -n staging \
  -o jsonpath='{.data}' | python3 -m json.tool | \
  python3 -c "import sys, json; print(sorted(json.load(sys.stdin).keys()))"

# Compare key sets (should be identical, values will differ)
```

## Conclusion

Portainer's ConfigMap list view provides a clean interface for browsing and searching configurations across namespaces. Use the search bar for quick filtering by name, and kubectl for advanced filtering by labels, annotations, and content. Regularly audit ConfigMaps to identify unused ones for cleanup, export them for backup before major changes, and compare configurations across namespaces to ensure consistency in your deployment pipeline.
