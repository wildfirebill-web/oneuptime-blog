# RKE2 vs Kubeadm: Kubernetes Installation Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubeadm, Kubernetes, Installation, Comparison

Description: A detailed comparison of RKE2 and kubeadm for bootstrapping Kubernetes clusters, covering ease of use, security, upgrades, and production readiness.

## Overview

RKE2 and kubeadm are two popular methods for bootstrapping Kubernetes clusters. Kubeadm is the official Kubernetes cluster bootstrap tool, providing a flexible but manual approach. RKE2 is a security-focused Kubernetes distribution from SUSE Rancher that includes kubeadm-like bootstrapping with added security defaults, lifecycle management, and Rancher integration. This guide compares them to help you choose the right installation method.

## What Is Kubeadm?

Kubeadm is the official Kubernetes tool for bootstrapping clusters. It handles the phases of cluster initialization (init) and node joining (join), but leaves infrastructure setup, OS configuration, networking, and lifecycle management to the operator. Kubeadm is the foundation of many other Kubernetes distributions including EKS, AKS, and GKE.

## What Is RKE2?

RKE2 is a fully opinionated Kubernetes distribution that handles cluster bootstrapping, security hardening, component management, and upgrades as a complete package. It is designed for production and enterprise environments, with CIS benchmark compliance enabled by default.

## Feature Comparison

| Feature | RKE2 | Kubeadm |
|---|---|---|
| Security Hardening | CIS defaults built-in | Manual configuration required |
| FIPS Support | Yes | No (manual) |
| Upgrade Path | Automated (systemd) | Manual (kubeadm upgrade) |
| etcd Management | Embedded and managed | External or embedded (manual) |
| Air-gap Support | Yes (tarball installer) | Manual (image pre-loading) |
| Rancher Integration | Native | Via import |
| Container Runtime | containerd (bundled) | containerd or Docker (manual) |
| CNI Plugin | Bundled (Canal/Calico/Cilium) | Manual installation |
| Certificates | Auto-rotated | Manual rotation |
| Single Binary | Yes | No (separate kubectl, kubeadm) |
| RBAC Default | Yes | Yes |
| Audit Logging | Configurable | Configurable |
| STIG Profile | Yes | No |
| CIS Profile | Yes | No |
| Backup/Restore (etcd) | Integrated | Manual |

## Cluster Installation

### Kubeadm

```bash
# Step 1: Install prerequisites on all nodes

apt-get update && apt-get install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl

# Step 2: Install container runtime (containerd)
apt-get install -y containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd

# Step 3: Initialize the control plane
kubeadm init --pod-network-cidr=10.244.0.0/16

# Step 4: Configure kubectl
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Step 5: Install CNI plugin (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Step 6: Join worker nodes
kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### RKE2

```bash
# Step 1: Install RKE2 server
curl -sfL https://get.rke2.io | sh -

# Step 2: Create config file
mkdir -p /etc/rancher/rke2
cat > /etc/rancher/rke2/config.yaml << 'EOF'
# Token for agent nodes to join
token: my-shared-secret

# TLS SANs for the API server
tls-san:
  - "192.168.1.100"
  - "rancher.example.com"

# CNI plugin (default: canal)
cni: calico
EOF

# Step 3: Start the server
systemctl enable --now rke2-server.service

# Step 4: Install on agent node
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

cat > /etc/rancher/rke2/config.yaml << 'EOF'
server: https://192.168.1.100:9345
token: my-shared-secret
EOF

systemctl enable --now rke2-agent.service
```

Notice that RKE2's installation is significantly simpler and self-contained. Containerd and CNI plugins are bundled.

## Upgrades

### Kubeadm Upgrade Process

```bash
# Upgrade kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-1.1

# Check upgrade plan
kubeadm upgrade plan

# Apply upgrade
kubeadm upgrade apply v1.29.0

# Upgrade kubelet and kubectl on each node
apt-get update && apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
systemctl daemon-reload && systemctl restart kubelet
```

### RKE2 Upgrade Process

```bash
# RKE2 can be upgraded with the Automated Upgrade Controller
# Or manually by updating the binary:
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.29.0+rke2r1 sh -
systemctl restart rke2-server
```

Alternatively, use the Rancher UI or System Upgrade Controller for zero-downtime rolling upgrades.

## Security Defaults

RKE2 applies CIS Kubernetes benchmark hardening by default, including:

- Pod Security Admission policies
- Audit logging configuration
- Restricted network policies
- etcd encryption at rest
- Disabled anonymous authentication

Kubeadm installs vanilla Kubernetes without security hardening. All of these settings must be manually configured.

## When to Choose RKE2

- Security hardening, FIPS, or CIS compliance is required
- You want reduced operational complexity
- Rancher management is planned
- You need integrated etcd backup and restore
- Your team prefers a complete, opinionated distribution

## When to Choose Kubeadm

- You want the most vanilla Kubernetes installation
- Deep customization of every component is required
- You want to learn how Kubernetes bootstrapping works
- Your organization has existing tooling built around kubeadm

## Conclusion

RKE2 and kubeadm both produce fully conformant Kubernetes clusters, but the path to get there is very different. Kubeadm gives you maximum control but requires significant manual work for security, networking, and upgrades. RKE2 trades some flexibility for a dramatically simplified operational experience with enterprise-grade security defaults. For most production use cases, especially those requiring compliance, RKE2 is the more practical choice.
