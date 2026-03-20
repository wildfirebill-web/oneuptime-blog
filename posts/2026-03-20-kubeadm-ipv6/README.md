# How to Initialize a Kubernetes Cluster with IPv6 Using kubeadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, kubeadm, Cluster Setup, Dual-Stack, Networking

Description: Initialize a dual-stack or IPv6-only Kubernetes cluster with kubeadm, configure pod and service CIDRs for IPv6, and set up the necessary node networking for IPv6 support.

## Introduction

kubeadm supports dual-stack and IPv6-only Kubernetes clusters through the `--pod-network-cidr` and `--service-cidr` flags. For dual-stack, provide both IPv4 and IPv6 CIDR ranges separated by commas. The control plane and nodes must have IPv6 enabled at the OS level, and the chosen CNI plugin must support IPv6. Calico and Cilium are the most commonly used CNI plugins with dual-stack support.

## Pre-requisites: Enable IPv6 on All Nodes

```bash
# Enable IPv6 on all control plane and worker nodes
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Persist settings
sudo tee /etc/sysctl.d/99-kubernetes-ipv6.conf << 'EOF'
net.ipv6.conf.all.disable_ipv6=0
net.ipv6.conf.default.disable_ipv6=0
net.ipv6.conf.all.forwarding=1
net.bridge.bridge-nf-call-ip6tables=1
EOF

sudo sysctl --system
```

## Initialize Dual-Stack Cluster

```bash
# kubeadm-config.yaml for dual-stack cluster
cat << 'EOF' > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
networking:
  podSubnet: "10.244.0.0/16,fd00:10:244::/56"
  serviceSubnet: "10.96.0.0/12,fd00:10:96::/108"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    node-ip: "192.168.1.10,2001:db8::10"
EOF

# Initialize cluster with dual-stack
sudo kubeadm init --config kubeadm-config.yaml

# Or inline with flags
sudo kubeadm init \
    --pod-network-cidr="10.244.0.0/16,fd00:10:244::/56" \
    --service-cidr="10.96.0.0/12,fd00:10:96::/108" \
    --apiserver-advertise-address=192.168.1.10

# Set up kubectl
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

## Initialize IPv6-Only Cluster

```yaml
# kubeadm-ipv6only.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
networking:
  podSubnet: "fd00:10:244::/56"
  serviceSubnet: "fd00:10:96::/108"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  kubeletExtraArgs:
    node-ip: "2001:db8::10"
localAPIEndpoint:
  advertiseAddress: "2001:db8::10"
  bindPort: 6443
```

```bash
sudo kubeadm init --config kubeadm-ipv6only.yaml
```

## Join Worker Nodes with IPv6

```bash
# Join command from kubeadm init output
# Specify node-ip for dual-stack workers
sudo kubeadm join 192.168.1.10:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --node-ip="192.168.1.11,2001:db8::11"

# Verify node joined with dual-stack
kubectl get nodes -o wide
# Should show both IPv4 and IPv6 under INTERNAL-IP
```

## Install CNI Plugin for IPv6

```bash
# Install Calico with dual-stack support
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

cat << 'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
      - blockSize: 122
        cidr: fd00:10:244::/56
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
EOF

# Verify Calico pods are running
kubectl -n calico-system get pods
```

## Verify Dual-Stack Cluster

```bash
# Check node addresses
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {range .status.addresses[*]}{.type}={.address} {end}{"\n"}{end}'

# Check cluster info
kubectl cluster-info

# Deploy a test pod and check IPv6 address
kubectl run test-pod --image=alpine --command -- sleep infinity
kubectl get pod test-pod -o jsonpath='{.status.podIPs}'
# Should show: [{"ip":"10.244.x.x"},{"ip":"fd00:10:244::x"}]

# Check service CIDR
kubectl get svc kubernetes -o jsonpath='{.spec.clusterIPs}'
# Should show: ["10.96.0.1","fd00:10:96::1"]
```

## Conclusion

Initialize a dual-stack Kubernetes cluster with kubeadm by providing comma-separated IPv4 and IPv6 CIDRs for both `podSubnet` and `serviceSubnet` in the kubeadm config. Enable IPv6 forwarding on all nodes before initialization. Specify `node-ip` with both IPv4 and IPv6 addresses for the kubelet on each node. Install a dual-stack capable CNI plugin (Calico or Cilium) configured with both IPv4 and IPv6 IP pools. Verify with `kubectl get nodes -o wide` and check pod IPs for dual-stack address assignment.
