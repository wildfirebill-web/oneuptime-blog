# How to Migrate from Kubernetes Dashboard to Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Migration, Dashboard, DevOps

Description: Migrate your Kubernetes workload management from the default Kubernetes Dashboard to Portainer for enhanced RBAC and multi-cluster support.

## Introduction

The Kubernetes Dashboard is the default web UI for Kubernetes clusters, but it has significant limitations: basic RBAC, no multi-cluster support, and limited deployment tools. Portainer provides a superior Kubernetes management experience with team access control, GitOps integration, and Helm support.

## Limitations of Kubernetes Dashboard

- No multi-cluster management from a single interface
- Limited user authentication (token-based only)
- No built-in team/role management
- No Helm chart deployment support
- No container template library
- No audit logging

## Deploying Portainer for Kubernetes

```bash
# Install Portainer on your Kubernetes cluster via Helm
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# Install in the portainer namespace
helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set tls.force=true

# Or with NodePort for on-premise clusters
helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --set service.type=NodePort \
  --set service.nodePort.https=30779

# Check deployment status
kubectl -n portainer get pods
kubectl -n portainer get svc
```

## Connecting Portainer to Your Existing Cluster

If running Portainer outside the cluster:

```bash
# Get the cluster's kubeconfig
cat ~/.kube/config

# In Portainer: Home > Add Environment > Kubernetes via kubeconfig
# Upload the kubeconfig file
```

For clusters with kubeconfig:

```bash
# Import the cluster kubeconfig via Portainer API
curl -X POST \
  -H "X-API-Key: your-api-key" \
  -F "KubeconfigFile=@/path/to/kubeconfig" \
  "https://portainer.example.com/api/endpoints/create/kubeconfig?name=my-cluster"
```

## Mapping Kubernetes Dashboard Features to Portainer

| Kubernetes Dashboard | Portainer Equivalent |
|---------------------|---------------------|
| Workloads > Deployments | Kubernetes > Applications |
| Services | Kubernetes > Services |
| Config Maps | Kubernetes > ConfigMaps |
| Secrets | Kubernetes > Secrets |
| Persistent Volume Claims | Kubernetes > Volumes |
| Namespaces | Kubernetes > Namespaces |
| Nodes | Kubernetes > Cluster > Nodes |
| Pod Logs | Applications > Pod > Logs |
| Pod Exec (Shell) | Applications > Pod > Console |
| Resource YAML edit | Applications > Edit YAML |

## Setting Up Namespace-Based Access Control

Portainer's team access is more powerful than the Dashboard's:

```bash
# In Portainer UI:
# 1. Settings > Users > Create users for your team
# 2. Settings > Teams > Create teams (e.g., "backend-team", "frontend-team")
# 3. Environments > Your Cluster > Namespaces
# 4. Assign team access to specific namespaces
```

## Migrating RBAC Configurations

```yaml
# Export existing ClusterRoleBindings and RoleBindings
kubectl get rolebindings,clusterrolebindings -A -o yaml > rbac-backup.yaml

# In Portainer, RBAC is managed at:
# - Environment level: read-only vs. read-write access
# - Namespace level: team-specific namespace access
# These replace the need for complex Kubernetes RBAC in many cases
```

## Using Helm in Portainer (Not Available in Dashboard)

```bash
# Add Helm repositories in Portainer
# Kubernetes > Helm Repositories > Add

# Deploy a Helm chart
# Kubernetes > Helm Charts > Browse > Deploy
# Configure values and deploy without CLI access
```

## Removing Kubernetes Dashboard After Migration

```bash
# Remove Kubernetes Dashboard after verifying Portainer works
kubectl delete namespace kubernetes-dashboard

# Or if installed via manifest
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Verify removal
kubectl get pods -n kubernetes-dashboard
```

## Accessing Multiple Clusters

One of Portainer's key advantages over the Dashboard:

```bash
# Add multiple clusters to Portainer
# Home > Add Environment (repeat for each cluster)
# Supported types:
# - Kubernetes via Agent
# - Kubernetes via kubeconfig
# - EKS, AKS, GKE via cloud

# Switch between clusters via the Portainer home screen
```

## Conclusion

Migrating from the Kubernetes Dashboard to Portainer provides multi-cluster management, proper team RBAC, Helm chart deployment, and GitOps integration—features absent from the default Dashboard. The migration is non-disruptive: Portainer sits alongside your existing cluster without changing workload configurations, making it a safe, incremental upgrade.
