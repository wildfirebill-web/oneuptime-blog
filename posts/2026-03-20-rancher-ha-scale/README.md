# How to Scale Rancher HA Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, High Availability, Scaling, Control Plane, etcd

Description: Scale your Rancher HA deployment by adding control plane nodes, scaling Rancher replicas, and separating etcd from control plane roles for improved performance.

## Introduction

As your Rancher deployment manages more clusters, you may need to scale the management infrastructure. This guide covers adding control plane nodes, scaling Rancher replicas, and optimizing node roles by separating etcd from control plane responsibilities.

## Prerequisites

- Running Rancher HA deployment
- New servers meeting minimum requirements
- Cluster admin access
- Maintenance window scheduled

## Step 1: Add a New Control Plane Node (RKE2)

```bash
# Prepare the new node

# On new-cp-node:
mkdir -p /etc/rancher/rke2

cat > /etc/rancher/rke2/config.yaml << EOF
# Same token as existing cluster
token: "my-cluster-token"
# Point to existing cluster
server: https://rancher.example.com:9345
tls-san:
  - rancher.example.com
  - 10.0.0.100
# Same roles as existing control plane nodes
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
EOF

# Install RKE2
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server
systemctl start rke2-server

# Monitor join progress
journalctl -u rke2-server -f &

# From management machine, verify new node joined
kubectl get nodes -w
```

## Step 2: Separate etcd from Control Plane

```bash
# For large deployments, separate etcd nodes from API server nodes
# This requires a fresh cluster or careful migration

# etcd-only node configuration
cat > /etc/rancher/rke2/config.yaml << EOF
token: "my-cluster-token"
server: https://existing-cp:9345

# Dedicated etcd role only
# Remove: control-plane role
# This node will only run etcd
disable-scheduler: true
disable-controller-manager: true
disable-etcd: false
EOF

# Control-plane only configuration (no etcd)
cat > /etc/rancher/rke2/config.yaml << EOF
token: "my-cluster-token"
server: https://existing-cp:9345

# Control plane role only, no etcd
disable-etcd: true
disable-scheduler: false
disable-controller-manager: false
EOF
```

## Step 3: Scale Rancher Replicas

```bash
# Scale Rancher to handle more load
kubectl scale deployment rancher \
  -n cattle-system \
  --replicas=5

# Or update via Helm (recommended for persistent configuration)
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set replicas=5 \
  --reuse-values

# Verify new replicas are running
kubectl get pods -n cattle-system -l app=rancher

# Ensure new pods are distributed across nodes
kubectl get pods -n cattle-system -o wide
```

## Step 4: Configure Pod Anti-Affinity for Distribution

```yaml
# rancher-affinity.yaml - Ensure Rancher pods spread across nodes
# Apply as Helm values or direct patch

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - rancher
        topologyKey: kubernetes.io/hostname
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - rancher
          topologyKey: topology.kubernetes.io/zone
```

```bash
# Apply affinity via Helm
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --reuse-values \
  --set affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].topologyKey=kubernetes.io/hostname
```

## Step 5: Add Worker Nodes for System Workloads

```bash
# Add dedicated worker nodes for cattle-system workloads
# (not for managed application workloads)

# Label and taint nodes for system use
kubectl taint nodes rancher-system-01 \
  rancher-system=true:NoSchedule

kubectl label nodes rancher-system-01 \
  rancher-system=true

# Update Rancher deployment to tolerate this taint
kubectl patch deployment rancher \
  -n cattle-system \
  --type=json \
  -p='[{
    "op": "add",
    "path": "/spec/template/spec/tolerations",
    "value": [{
      "key": "rancher-system",
      "operator": "Equal",
      "value": "true",
      "effect": "NoSchedule"
    }]
  }]'
```

## Step 6: Scale Fleet Agent for GitOps

```bash
# Scale fleet components for large-scale GitOps
kubectl scale deployment fleet-controller \
  -n cattle-fleet-system \
  --replicas=2

kubectl scale deployment fleet-gitjob \
  -n cattle-fleet-system \
  --replicas=2

# Increase Fleet controller resources
kubectl patch deployment fleet-controller \
  -n cattle-fleet-system \
  --type=json \
  -p='[{
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources",
    "value": {
      "requests": {"cpu": "500m", "memory": "512Mi"},
      "limits": {"cpu": "2000m", "memory": "2Gi"}
    }
  }]'
```

## Step 7: Validate Scaling

```bash
# Verify all components are healthy after scaling

# Check etcd cluster
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
  -w table

# Check Rancher replicas
kubectl get deployment rancher -n cattle-system

# Check all pods distributed
kubectl get pods -n cattle-system -o wide

# Test Rancher UI responsiveness
time curl -sk https://rancher.example.com/v3/clusters
```

## Conclusion

Scaling Rancher HA requires careful planning and execution. Adding control plane nodes provides more API server capacity and etcd resilience, while scaling Rancher replicas distributes UI and API traffic. For deployments managing 50+ clusters, separating etcd and control plane roles onto dedicated nodes prevents resource contention between etcd (I/O intensive) and the Kubernetes API server (CPU/memory intensive). Always verify pod anti-affinity rules ensure replicas are distributed across failure domains.
