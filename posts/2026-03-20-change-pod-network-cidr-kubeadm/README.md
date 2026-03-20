# How to Change the Default Pod Network CIDR for IPv4 in kubeadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, kubeadm, IPv4, Pod CIDR, CNI, Networking

Description: Configure a custom IPv4 Pod CIDR when initializing a Kubernetes cluster with kubeadm to avoid conflicts with existing network ranges.

The default pod CIDR varies by CNI plugin (Flannel uses 10.244.0.0/16, Calico uses 192.168.0.0/16). In environments where these overlap with existing infrastructure, you must specify a custom CIDR during kubeadm init.

## Why Change the Default Pod CIDR?

- Your on-premises network already uses `10.244.0.0/16`
- Corporate policy requires a different private range
- You're connecting multiple clusters and need non-overlapping pod CIDRs
- You need a larger/smaller allocation than the default

## Method 1: kubeadm init Config File (Recommended)

```yaml
# kubeadm-init-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "v1.29.0"
networking:
  # Custom pod CIDR - avoids overlap with 10.x corporate networks
  podSubnet: "172.16.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
```

```bash
# Initialize with custom CIDR
sudo kubeadm init --config kubeadm-init-config.yaml

# Save the kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Method 2: Command Line Flag

```bash
# Quick single-node setup with custom CIDR
sudo kubeadm init \
  --pod-network-cidr=172.16.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.29.0
```

## Step 2: Install CNI with Matching CIDR

After init, install a CNI plugin and configure it to use the same CIDR.

**Flannel with custom CIDR:**

```bash
# Download the Flannel manifest
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Edit the Network field to match your pod CIDR
sed -i 's|10.244.0.0/16|172.16.0.0/16|g' kube-flannel.yml

# Apply the modified manifest
kubectl apply -f kube-flannel.yml
```

**Calico with custom CIDR:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Patch the Calico config to use your CIDR
kubectl set env daemonset/calico-node -n kube-system \
  CALICO_IPV4POOL_CIDR="172.16.0.0/16"
```

## Verifying the CIDR is Applied

```bash
# Check that nodes receive the correct per-node CIDR
kubectl get nodes -o wide
kubectl describe node <node-name> | grep PodCIDR

# Test with a pod
kubectl run test --image=busybox --restart=Never -- sleep 3600
kubectl get pod test -o wide
# Pod IP should be in 172.16.x.x range
```

## Checking for CIDR Conflicts

```bash
# Ensure the pod CIDR doesn't overlap with host routing
ip route | grep -v "wg\|tun\|docker"
# None of these routes should overlap with 172.16.0.0/16
```

Always verify the custom CIDR doesn't conflict with existing routes before cluster creation, as changing the CIDR post-init requires a full cluster rebuild.
