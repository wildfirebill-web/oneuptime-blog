# How to Configure Cross-Cluster Service Discovery in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Networking, Service Discovery, Submariner

Description: Configure cross-cluster service discovery in Rancher-managed clusters using Submariner to enable pods in one cluster to reach services in another using standard Kubernetes DNS.

## Introduction

By default, Kubernetes services are only resolvable within their own cluster. Cross-cluster service discovery bridges this gap, allowing a microservice in cluster A to call a database service in cluster B using a consistent DNS name. This guide covers implementing cross-cluster service discovery using Submariner, a SUSE-backed project designed for Rancher environments.

## Prerequisites

- Rancher managing at least two clusters
- Non-overlapping Pod and Service CIDR ranges across clusters
- UDP port 4500 open between cluster nodes (for Submariner's VXLAN/IPsec tunnel)

## Understanding Submariner

Submariner creates a secure tunnel between clusters and installs a DNS proxy that resolves `<service>.<namespace>.svc.clusterset.local` to the correct endpoint in any joined cluster.

## Step 1: Verify Non-Overlapping CIDRs

```bash
# Check Pod CIDR for each cluster

for kubeconfig in /tmp/kc-cluster-*.yaml; do
  echo "=== ${kubeconfig} ==="
  kubectl --kubeconfig="${kubeconfig}" get nodes \
    -o jsonpath='{.items[*].spec.podCIDR}' && echo
done

# Example safe configuration:
# Cluster 1: pods=10.42.0.0/16,  services=10.43.0.0/16
# Cluster 2: pods=10.44.0.0/16,  services=10.45.0.0/16
# Cluster 3: pods=10.46.0.0/16,  services=10.47.0.0/16
```

## Step 2: Install subctl CLI

```bash
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:~/.local/bin
subctl version
```

## Step 3: Deploy the Submariner Broker

The broker is a coordination component installed on one cluster (typically the Rancher management cluster):

```bash
# Install the broker on the hub/management cluster
subctl deploy-broker \
  --kubeconfig /tmp/kc-management.yaml \
  --globalnet   # Enable if CIDRs do overlap (uses virtual IPs)

# A broker-info.subm file is generated - this is needed to join other clusters
ls broker-info.subm
```

## Step 4: Join Clusters to Submariner

```bash
# Join cluster-1
subctl join broker-info.subm \
  --kubeconfig /tmp/kc-cluster-1.yaml \
  --clusterid cluster-1 \
  --natt=false      # Set true if behind NAT

# Join cluster-2
subctl join broker-info.subm \
  --kubeconfig /tmp/kc-cluster-2.yaml \
  --clusterid cluster-2 \
  --natt=false

# Verify the connection
subctl show connections
# Expected:
# Cluster cluster-1 → cluster-2: connected
# Cluster cluster-2 → cluster-1: connected
```

## Step 5: Verify Cluster Connectivity

```bash
# Check Submariner pods
kubectl --kubeconfig /tmp/kc-cluster-1.yaml \
  get pods -n submariner-operator

# Run connectivity diagnostics
subctl diagnose all \
  --kubeconfig /tmp/kc-cluster-1.yaml

# Test pod-to-pod connectivity across clusters
subctl benchmark latency \
  --kubeconfig /tmp/kc-cluster-1.yaml \
  --remoteconfig /tmp/kc-cluster-2.yaml \
  --intra-cluster
```

## Step 6: Export a Service for Cross-Cluster Access

Services must be explicitly exported to be accessible from other clusters:

```bash
# Export the database service from cluster-1
kubectl --kubeconfig /tmp/kc-cluster-1.yaml \
  annotate service postgres \
  -n production \
  "submariner.io/export=true"

# Or create a ServiceExport resource directly:
kubectl --kubeconfig /tmp/kc-cluster-1.yaml \
  apply -f - << 'EOF'
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: postgres
  namespace: production
EOF
```

## Step 7: Consume the Service from Another Cluster

```bash
# Verify the service is visible from cluster-2
kubectl --kubeconfig /tmp/kc-cluster-2.yaml \
  get serviceimport -n production

# Test DNS resolution from a pod in cluster-2
kubectl --kubeconfig /tmp/kc-cluster-2.yaml \
  run dns-test --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- nslookup postgres.production.svc.clusterset.local

# The DNS name format:
# <service>.<namespace>.svc.clusterset.local
# ↑ Note: "clusterset.local" instead of "cluster.local"
```

## Step 8: Use the Cross-Cluster Service in Your Application

```yaml
# Deployment in cluster-2 that uses a service from cluster-1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: ghcr.io/my-org/myapp:latest
          env:
            - name: DATABASE_URL
              # Cross-cluster DNS name - resolves to postgres in cluster-1
              value: "postgres://user:pass@postgres.production.svc.clusterset.local:5432/mydb"
```

## Step 9: Configure Headless Service Export (StatefulSets)

```yaml
# Export a headless service (for StatefulSets with direct pod addressing)
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: postgres-headless
  namespace: production
# Individual pod DNS: postgres-0.postgres-headless.production.svc.clusterset.local
```

## Step 10: Monitor Submariner Health

```bash
# Check tunnel status
subctl show connections

# Check endpoint resources
kubectl get endpoint -n submariner-operator

# View Submariner metrics in Prometheus
kubectl port-forward -n submariner-operator \
  service/submariner-metrics 9898:9898
curl http://localhost:9898/metrics | grep submariner
```

## Conclusion

Cross-cluster service discovery with Submariner transforms isolated Kubernetes clusters into an interconnected service mesh. By exporting services with `ServiceExport` resources and consuming them via the `clusterset.local` DNS domain, applications can span multiple Rancher-managed clusters transparently. This enables powerful patterns like centralizing shared services (databases, message queues) in one cluster while application workloads run in separate clusters with full connectivity.
