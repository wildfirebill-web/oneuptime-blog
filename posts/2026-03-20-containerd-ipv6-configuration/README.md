# How to Configure containerd with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: containerd, IPv6, CNI, Container Runtime, Kubernetes, Networking

Description: A guide to configuring containerd container runtime with IPv6 and dual-stack networking, including CNI plugin configuration and Kubernetes integration.

containerd is the industry-standard container runtime used by Kubernetes. IPv6 configuration for containerd is handled through CNI (Container Network Interface) plugins. This guide covers setting up containerd with IPv6-capable CNI plugins for both standalone use and Kubernetes deployments.

## containerd Configuration Overview

containerd itself does not manage IP addressing — that is delegated to CNI plugins. The containerd configuration (`/etc/containerd/config.toml`) specifies CNI plugin paths:

```toml
# /etc/containerd/config.toml

version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".cni]
    bin_dir = "/opt/cni/bin"
    conf_dir = "/etc/cni/net.d"
    # containerd will use the CNI configs in conf_dir for pod networking
```

## Installing CNI Plugins

```bash
# Download CNI plugins
CNI_VERSION="v1.4.0"
curl -LO https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz

# Install
mkdir -p /opt/cni/bin
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-${CNI_VERSION}.tgz

# Verify
ls /opt/cni/bin/ | grep -E "bridge|host-local|portmap|loopback"
```

## CNI Configuration with IPv6

```json
// /etc/cni/net.d/10-containerd-net.conflist

{
  "cniVersion": "1.0.0",
  "name": "containerd-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "promiscMode": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.22.0.0/16"}],
          [{"subnet": "fd00:containerd::/64"}]
        ],
        "routes": [
          {"dst": "0.0.0.0/0"},
          {"dst": "::/0"}
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    },
    {
      "type": "loopback"
    }
  ]
}
```

## Enable IPv6 in Kernel for containerd

```bash
# Enable IPv6 forwarding (required for container networking)
cat >> /etc/sysctl.d/99-containerd-ipv6.conf << 'EOF'
net.ipv6.conf.all.forwarding = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# Load br_netfilter module
modprobe br_netfilter
echo br_netfilter >> /etc/modules-load.d/k8s.conf
```

## Running Containers with IPv6 via nerdctl

nerdctl is a Docker-compatible CLI for containerd:

```bash
# Install nerdctl
VERSION="1.7.3"
curl -LO https://github.com/containerd/nerdctl/releases/download/v${VERSION}/nerdctl-${VERSION}-linux-amd64.tar.gz
tar -C /usr/local/bin -xzf nerdctl-${VERSION}-linux-amd64.tar.gz

# Run a container using the IPv6-enabled CNI network
nerdctl run -d \
  --name web \
  --network containerd-net \
  -p 80:80 \
  nginx:alpine

# Check container IPv6 address
nerdctl inspect web | python3 -m json.tool | grep -A 5 "IPv6\|GlobalIPv6"

# Test IPv6 connectivity
nerdctl exec web ip -6 addr show
nerdctl exec web curl -6 https://ipv6.google.com
```

## Kubernetes with containerd Dual-Stack

When using containerd with Kubernetes, the dual-stack configuration is primarily in kube-apiserver and kubelet:

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  serviceSubnet: "10.96.0.0/16,fd00:service::/108"
  podSubnet: "10.244.0.0/16,fd00:pod::/48"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
```

```bash
# Initialize cluster with dual-stack
kubeadm init --config kubeadm-config.yaml

# Verify pod gets dual-stack addresses
kubectl get pod -o wide
kubectl exec <pod> -- ip -6 addr show
```

## Calico CNI with containerd for IPv6

Calico is a popular CNI that works well with containerd for IPv6:

```yaml
# calico-config.yaml (relevant IPv6 sections)
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv6-ippool
spec:
  cidr: fd00:pod::/48
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
```

```bash
# Apply Calico with IPv6 enabled
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Set CALICO_IPV6POOL_CIDR environment variable in calico-node DaemonSet
kubectl set env daemonset/calico-node \
  -n kube-system \
  IP6=autodetect \
  CALICO_IPV6POOL_CIDR=fd00:pod::/48
```

## Verifying containerd IPv6 Setup

```bash
# Check containerd service is running
systemctl status containerd

# List running containers
nerdctl ps

# Check CNI configuration is loaded
ls -la /etc/cni/net.d/

# Verify IPv6 bridge was created
ip -6 addr show cni0

# Check IPv6 routes for container traffic
ip -6 route show | grep cni0

# Test container DNS with IPv6
nerdctl exec web nslookup -type=AAAA ipv6.google.com
```

containerd's IPv6 support is entirely CNI-driven, making it flexible: configure the appropriate CNI plugin (bridge for standalone, Calico/Cilium/Flannel for Kubernetes) with IPv6 subnets, and containerd containers automatically receive dual-stack addresses.
