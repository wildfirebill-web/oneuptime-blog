# How to Implement Cluster Federation with Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Federation, Multi-Cluster

Description: Implement Kubernetes cluster federation with Rancher using KubeFed or Fleet to synchronize resources and policies across multiple clusters.

## Introduction

Cluster federation enables you to manage resources across multiple Kubernetes clusters as if they were a single unified cluster. Use cases include global application deployments, disaster recovery, and policy enforcement across cluster boundaries. This guide covers implementing federation using both Rancher Fleet (for policy and config) and KubeFed (for workload federation).

## Approaches to Federation in Rancher

| Approach | Best For | Component |
|---|---|---|
| **Rancher Fleet** | GitOps-driven config/policy sync | Built into Rancher |
| **KubeFed** | Workload federation with failover | Open-source KubeFed |
| **Submariner** | Cross-cluster service discovery | SUSE-backed project |

This guide covers all three.

## Part 1: Fleet-Based Federation

Fleet provides the simplest form of federation - syncing Kubernetes resources from Git to multiple clusters simultaneously.

### Create a Federated Application

```yaml
# fleet-federation/gitrepo.yaml

apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: federated-app
  namespace: fleet-default
spec:
  repo: https://github.com/my-org/federated-app
  branch: main
  paths:
    - deploy/
  targets:
    # Deploy to all production clusters in any region
    - name: all-production
      clusterSelector:
        matchLabels:
          environment: production
      # Override replica counts per cluster
      helm:
        values:
          replicaCount: 3
```

## Part 2: KubeFed for Workload Federation

KubeFed provides richer federation semantics: cross-cluster placement, replica scheduling, and status aggregation.

### Install KubeFed

```bash
# Install KubeFed on the Rancher management cluster
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm repo update

helm install kubefed kubefed-charts/kubefed \
  --namespace kube-federation-system \
  --create-namespace
```

### Join Clusters to the Federation

```bash
# Install kubefedctl CLI
curl -LO https://github.com/kubernetes-sigs/kubefed/releases/download/v0.10.0/kubefedctl-0.10.0-linux-amd64.tgz
tar xzf kubefedctl-0.10.0-linux-amd64.tgz
sudo mv kubefedctl /usr/local/bin/

# Join downstream clusters
kubefedctl join cluster-us-east \
  --cluster-context rancher-prod-us \
  --host-cluster-context rancher-management \
  --v=2

kubefedctl join cluster-eu-west \
  --cluster-context rancher-prod-eu \
  --host-cluster-context rancher-management \
  --v=2

# Verify
kubectl get kubefedcluster -n kube-federation-system
```

### Create a Federated Deployment

```yaml
# federated-deployment.yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: myapp
  namespace: production
spec:
  # Base template (same as a regular Deployment)
  template:
    metadata:
      labels:
        app: myapp
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: myapp
      template:
        metadata:
          labels:
            app: myapp
        spec:
          containers:
            - name: myapp
              image: ghcr.io/my-org/myapp:1.0.0
              ports:
                - containerPort: 8080

  # Placement - which clusters to deploy to
  placement:
    clusters:
      - name: cluster-us-east
      - name: cluster-eu-west

  # Overrides - cluster-specific customizations
  overrides:
    - clusterName: cluster-us-east
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5           # More replicas in US
    - clusterName: cluster-eu-west
      clusterOverrides:
        - path: "/spec/replicas"
          value: 2           # Fewer replicas in EU (lower traffic)
```

### Create a ReplicaSchedulingPreference

```yaml
# rsp.yaml - automatically distribute replicas based on cluster weight
apiVersion: scheduling.kubefed.io/v1alpha1
kind: ReplicaSchedulingPreference
metadata:
  name: myapp
  namespace: production
spec:
  targetKind: FederatedDeployment
  targetName: myapp
  totalReplicas: 10
  clusters:
    cluster-us-east:
      weight: 60    # 60% of replicas in US
    cluster-eu-west:
      weight: 40    # 40% of replicas in EU
```

## Part 3: Submariner for Cross-Cluster Service Discovery

Submariner enables services in one cluster to communicate with services in another cluster using their Kubernetes DNS names.

```bash
# Install Submariner using the operator
helm repo add submariner-latest https://submariner-io.github.io/submariner-charts/charts
helm repo update

# Install the broker on the hub cluster
helm install submariner-broker submariner-latest/submariner-operator \
  --namespace submariner-operator \
  --create-namespace \
  --set broker.server=true

# Join cluster-us-east
subctl join broker-info.subm --kubecontext rancher-prod-us \
  --clusterid cluster-us-east \
  --natt=false

# Join cluster-eu-west
subctl join broker-info.subm --kubecontext rancher-prod-eu \
  --clusterid cluster-eu-west \
  --natt=false
```

```yaml
# Export a service across clusters
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: myapp-database
  namespace: production
# Now myapp-database.production.svc.clusterset.local resolves in all joined clusters
```

## Monitoring Federation Health

```bash
# Check KubeFed cluster status
kubectl get kubefedcluster -n kube-federation-system

# Check Federated resource propagation
kubectl get federateddeployment -n production -o json \
  | jq '.items[].status.clusters[] | {cluster: .name, ready: .readyReplicas}'

# Check Submariner connectivity
subctl show connections
```

## Conclusion

Cluster federation with Rancher spans multiple levels of sophistication: Fleet provides simple, GitOps-driven resource synchronization; KubeFed adds intelligent placement and cross-cluster replica scheduling; and Submariner enables cross-cluster service discovery. Choose the approach that matches your complexity requirements - most organizations start with Fleet and add KubeFed or Submariner as their multi-cluster architecture matures.
