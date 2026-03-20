# How to Expand the Pod CIDR Range for IPv4 in an Existing Kubernetes Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv4, Pod CIDR, IPAM, Networking, Calico

Description: Expand the available IPv4 address space for pods in a running Kubernetes cluster by adding secondary IP pools without disrupting existing workloads.

Changing the primary Pod CIDR in an existing cluster is extremely disruptive and generally not supported. The correct approach is to add a secondary IP pool alongside the existing one, giving new pods access to the expanded address space.

## Why You Can't Simply Change the Existing CIDR

The Pod CIDR is embedded in:
- `kube-controller-manager --cluster-cidr` flag
- Node object `spec.podCIDR` field
- CNI plugin configuration
- Existing iptables/IPVS rules

Changing these requires a full cluster rebuild for most components.

## Option 1: Add a Secondary IP Pool in Calico

Calico's IPAM supports multiple IP pools natively:

```yaml
# secondary-pool.yaml

apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: secondary-ipv4-pool
spec:
  # New CIDR alongside the existing one
  cidr: 10.245.0.0/16
  vxlanMode: CrossSubnet
  ipipMode: Never
  natOutgoing: true
  disabled: false
  blockSize: 26
```

```bash
kubectl apply -f secondary-pool.yaml

# Verify both pools exist
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get ippool
```

New pods will automatically draw from whichever pool has available addresses.

## Option 2: Add New CIDR Allocation to Controller Manager

For clusters where the CNI doesn't manage IPAM independently:

```bash
# Step 1: Update the kube-controller-manager to include the new CIDR
sudo nano /etc/kubernetes/manifests/kube-controller-manager.yaml

# Change --cluster-cidr from:
# --cluster-cidr=10.244.0.0/16
# To (comma-separated for multiple CIDRs):
# --cluster-cidr=10.244.0.0/16,10.245.0.0/16
```

```bash
# Step 2: Update the CNI DaemonSet to use the expanded CIDR
# For Flannel: update the ConfigMap
kubectl edit configmap kube-flannel-cfg -n kube-flannel
# Change "Network" to include both CIDRs (or use the larger supernet)
```

## Option 3: Migrate to a Larger Supernet (Planned Maintenance)

If you need to eventually use a single larger CIDR, plan a gradual migration:

```bash
# 1. Disable the old pool (stop new allocations)
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl patch ippool default-ipv4-ippool \
  --patch '{"spec": {"disabled": true}}'

# 2. Enable the new larger pool
kubectl apply -f - << 'EOF'
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: expanded-ipv4-pool
spec:
  cidr: 10.244.0.0/14
  vxlanMode: CrossSubnet
  natOutgoing: true
  blockSize: 24
EOF

# 3. Gradually drain and recreate pods to get new IPs
kubectl rollout restart deployment/my-app -n production
```

## Verification

```bash
# Check both pools are active
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl ipam show --summary

# Deploy a test pod and verify it gets an IP from the new pool
kubectl run test-expansion --image=alpine --restart=Never -- sleep 3600
kubectl get pod test-expansion -o wide
# IP should be in 10.245.x.x range (if primary was exhausted)
```

Adding secondary pools is the least risky expansion strategy as it doesn't require any downtime or component restarts.
