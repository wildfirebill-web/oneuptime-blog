# How to Configure Pod CIDR Range for IPv4 in a Kubernetes Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv4, Pod CIDR, CNI, Networking, kubeadm

Description: Set the IPv4 Pod CIDR range during Kubernetes cluster initialization and understand how it is allocated across nodes by the controller manager.

The Pod CIDR defines the IPv4 address pool from which Kubernetes assigns addresses to pods. It must not overlap with host network ranges or service CIDRs, and must be large enough to accommodate the expected number of pods.

## Planning the Pod CIDR

```
Recommended: 10.244.0.0/16 (default for Flannel)
Alternative: 192.168.0.0/16 (default for Calico)
Custom:      10.100.0.0/16 (use any private range not in use)

With /16 Pod CIDR and /24 per-node allocation:
- Total addresses: 65,536
- Addresses per node: 256 (minus reserved = ~254 pods per node)
- Maximum nodes: 256
```

## Setting Pod CIDR with kubeadm

```bash
# Create a kubeadm configuration file
cat > kubeadm-config.yaml << 'EOF'
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  # IPv4 CIDR for pod addresses
  podSubnet: "10.244.0.0/16"
  # IPv4 CIDR for service ClusterIP addresses
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
EOF

# Initialize the cluster with the configuration
sudo kubeadm init --config kubeadm-config.yaml
```

Or pass directly via flags:

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12
```

## Verifying the Pod CIDR

```bash
# Check the cluster configuration
kubectl get cm kubeadm-config -n kube-system -o yaml | grep -A5 networking

# Check kube-controller-manager arguments
kubectl get pod -n kube-system kube-controller-manager-<node> -o yaml | \
  grep -i cidr

# View node CIDR allocations
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

## Per-Node CIDR Allocation

The kube-controller-manager automatically carves the Pod CIDR into per-node blocks:

```bash
# Check what CIDR is assigned to a specific node
kubectl get node worker-1 -o jsonpath='{.spec.podCIDR}'
# Expected: 10.244.1.0/24

kubectl get node worker-2 -o jsonpath='{.spec.podCIDR}'
# Expected: 10.244.2.0/24
```

## What Happens After cluster init

After setting the Pod CIDR, install a CNI plugin that understands it:

```bash
# For Flannel (uses podCIDR automatically from node spec)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# For Calico (specify the pod CIDR in the Calico config)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
# Then configure the CALICO_IPV4POOL_CIDR env var to match your podSubnet
```

## Verifying Pod IP Assignment

```bash
# Deploy a test pod and check its IP
kubectl run test-pod --image=alpine --restart=Never -- sleep 3600

kubectl get pod test-pod -o wide
# The pod should have an IP within the podSubnet range
# e.g., 10.244.1.5

# All pod IPs should be within the CIDR
kubectl get pods --all-namespaces -o wide | awk '{print $7}' | sort -u
```

Choosing the correct Pod CIDR at cluster creation time is important — changing it later requires significant effort and potential downtime.
