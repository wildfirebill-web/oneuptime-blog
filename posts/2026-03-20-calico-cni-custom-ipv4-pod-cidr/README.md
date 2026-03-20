# How to Install Calico CNI with a Custom IPv4 Pod CIDR

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Calico, Kubernetes, IPv4, CNI, Pod CIDR, Networking

Description: Install Calico CNI on a Kubernetes cluster and configure it to use a custom IPv4 CIDR pool for pod address allocation.

Calico is a popular CNI plugin that provides networking and network policy for Kubernetes. By default it uses `192.168.0.0/16` for pod IPs — here's how to customize that to match your cluster's Pod CIDR.

## Prerequisites

```bash
# Verify kubeadm was initialized with a matching Pod CIDR
kubectl cluster-info dump | grep cluster-cidr
# or check controller-manager flags
kubectl get pod -n kube-system kube-controller-manager-<node> -o yaml | grep cidr
```

## Method 1: Install Calico with a Custom CIDR via Operator

```bash
# Install the Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Create custom installation config with your CIDR
cat > calico-installation.yaml << 'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      # Set to match your kubeadm --pod-network-cidr
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
      blockSize: 26
EOF

kubectl create -f calico-installation.yaml
```

## Method 2: Install Calico with Manifests and Edit Directly

```bash
# Download the Calico manifest
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Edit the CALICO_IPV4POOL_CIDR environment variable
# Find and change: "192.168.0.0/16" to your custom CIDR
sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|' calico.yaml
sed -i 's|#   value: "192.168.0.0/16"|  value: "10.244.0.0/16"|' calico.yaml

# Apply the modified manifest
kubectl apply -f calico.yaml
```

## Verifying the IP Pool Configuration

```bash
# Install calicoctl
curl -L https://github.com/projectcalico/calico/releases/download/v3.27.0/calicoctl-linux-amd64 \
  -o calicoctl
chmod +x calicoctl && sudo mv calicoctl /usr/local/bin/

# View IP pools
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get ippool -o yaml

# Expected output:
# spec:
#   cidr: 10.244.0.0/16
#   ipipMode: Never
#   vxlanMode: CrossSubnet
#   natOutgoing: true
```

## Modifying an Existing IP Pool

```bash
# Edit the existing default IP pool
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get ippool default-ipv4-ippool -o yaml > ippool.yaml

# Edit ippool.yaml, change the cidr field, then apply
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl apply -f ippool.yaml
```

## Verifying Pod IP Assignment

```bash
# Check that pods receive IPs from the custom CIDR
kubectl get pods --all-namespaces -o wide | awk '{print $7}' | sort | head -20

# All IPs should be in the 10.244.0.0/16 range
kubectl run test-calico --image=alpine --restart=Never -- sleep 3600
kubectl get pod test-calico -o wide
# Expected IP: 10.244.x.x

# View Calico node IP block assignments
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl get ipamblock
```

## Check Calico System Status

```bash
# Verify all Calico pods are running
kubectl get pods -n calico-system
kubectl get pods -n kube-system | grep calico

# Check calico-node DaemonSet is ready on all nodes
kubectl rollout status daemonset/calico-node -n calico-system
```

The Calico `blockSize` (default `/26`) controls how the pool CIDR is subdivided per node — adjust it if you need more or fewer pod addresses per node.
