# How to Migrate from Rancher to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Rancher, Migration, Kubernetes, Docker

Description: Guide to migrating container and Kubernetes workload management from Rancher to Portainer for a lighter-weight management solution.

## Introduction

Rancher is a full-featured Kubernetes management platform, but its complexity and resource requirements can be overkill for teams managing a small number of clusters. Portainer provides a lighter-weight alternative with a simpler UI, faster setup, and lower overhead while still supporting Kubernetes, Docker, and edge deployments.

## When to Migrate from Rancher to Portainer

- You manage fewer than 10 clusters
- Your team finds Rancher's UI overly complex
- You need Docker (not just Kubernetes) management
- You want lower resource overhead
- You prefer a simpler RBAC model

## Exporting Rancher Workloads

```bash
# Export all namespaces and workloads from Rancher-managed clusters
# Using kubectl with Rancher kubeconfig

# Get all deployments
kubectl get deployments -A -o yaml > all-deployments.yaml

# Get all services
kubectl get services -A -o yaml > all-services.yaml

# Get all configmaps
kubectl get configmaps -A -o yaml > all-configmaps.yaml

# Get all persistent volume claims
kubectl get pvc -A -o yaml > all-pvcs.yaml

# Export Helm releases
helm list -A > helm-releases.txt
helm list -A -o json > helm-releases.json
```

## Documenting Rancher Projects and Namespaces

```bash
# List all Rancher projects (Rancher-specific CRD)
kubectl get projects.management.cattle.io -A

# List namespaces with their project annotations
kubectl get namespaces -o json | python3 -c "
import sys, json
ns_list = json.load(sys.stdin)
for ns in ns_list['items']:
    name = ns['metadata']['name']
    project = ns['metadata'].get('annotations', {}).get('field.cattle.io/projectId', 'none')
    print(f'{name}: project={project}')
"
```

## Setting Up Portainer for Kubernetes

```bash
# Deploy Portainer via Helm on your cluster
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --set service.type=LoadBalancer

# Wait for Portainer to be ready
kubectl -n portainer wait --for=condition=Ready pod -l app.kubernetes.io/name=portainer --timeout=300s

# Get the external IP
kubectl -n portainer get svc portainer
```

## Recreating Rancher Projects as Portainer Namespaces

Rancher's "Project" concept maps to Portainer's namespace-based access control:

```bash
# In Portainer UI:
# 1. Kubernetes > Namespaces > Create namespace
# 2. Create namespaces matching your Rancher project structure
# 3. Settings > Teams > Create teams matching Rancher users/groups
# 4. Assign teams to namespaces with appropriate roles

# Example namespace structure
kubectl create namespace production-backend
kubectl create namespace production-frontend
kubectl create namespace staging
```

## Migrating Helm Releases

```bash
# Helm releases are independent of Rancher
# They continue working after Rancher removal
helm list -A

# In Portainer: Kubernetes > Helm Charts
# Portainer shows all existing Helm releases
# You can manage upgrades/rollbacks from the UI
```

## Removing Rancher

Only after verifying Portainer manages all workloads:

```bash
# Uninstall Rancher (run on the Rancher host, not the managed cluster)
helm uninstall rancher -n cattle-system

# Remove Rancher-specific CRDs
kubectl get crd | grep cattle.io | awk '{print $1}' | xargs kubectl delete crd

# Remove Rancher system namespaces
kubectl delete namespace cattle-system cattle-global-data cattle-impersonation-system
```

## RBAC Migration

| Rancher Concept | Portainer Equivalent |
|----------------|---------------------|
| Project | Namespace group |
| Project Member | Team namespace access |
| Cluster Owner | Environment Admin |
| Cluster Member | Environment Standard User |
| Read-Only | Read-Only role |

## Conclusion

Migrating from Rancher to Portainer simplifies your Kubernetes management stack while preserving all workloads. The migration involves documenting current configurations, deploying Portainer alongside Rancher, recreating access control policies, and then removing Rancher once the transition is verified. Portainer's lighter footprint and unified Docker+Kubernetes management make it an excellent choice for smaller teams.
