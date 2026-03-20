# How to Install K3s on Fedora

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Fedora, Kubernetes, Linux, Installation, Lightweight Kubernetes, SUSE Rancher

Description: Learn how to install K3s on Fedora Linux, handle Fedora-specific requirements like firewall and SELinux configuration, and get a single-node or multi-node cluster running.

---

K3s is a lightweight Kubernetes distribution that installs in seconds. Fedora's SELinux enforcement and firewall require a few extra steps compared to other distributions.

---

## Prerequisites

- Fedora 38+ (K3s supports Fedora 36+)
- Minimum 2 CPU cores, 2GB RAM
- Root or sudo access

---

## Step 1: Prepare the System

```bash
# Update the system
sudo dnf upgrade -y

# Install required packages
sudo dnf install -y container-selinux selinux-policy-base

# Install K3s SELinux policy (required on Fedora with SELinux enforcing)
sudo dnf install -y https://rpm.rancher.io/k3s/stable/common/centos/8/noarch/k3s-selinux-1.4.1-1.el8.noarch.rpm
```

---

## Step 2: Configure the Firewall

Open the ports K3s needs:

```bash
# K3s API server
sudo firewall-cmd --permanent --add-port=6443/tcp

# Flannel VXLAN (if using default CNI)
sudo firewall-cmd --permanent --add-port=8472/udp

# Kubelet metrics
sudo firewall-cmd --permanent --add-port=10250/tcp

# NodePort range
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/udp

# Apply changes
sudo firewall-cmd --reload
```

---

## Step 3: Install K3s

```bash
# Install K3s server (single-node or first node in HA)
curl -sfL https://get.k3s.io | sh -

# Verify the service is running
sudo systemctl status k3s

# Check that the node is Ready
sudo kubectl get nodes
```

The installer automatically:
- Downloads the K3s binary
- Creates a systemd service
- Writes the kubeconfig to `/etc/rancher/k3s/k3s.yaml`

---

## Step 4: Configure kubectl Access for Non-Root Users

```bash
# Copy kubeconfig for your user
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Test access
kubectl get nodes
kubectl get pods -A
```

---

## Step 5: Add Agent Nodes (Optional)

Get the server token and add worker nodes:

```bash
# On the server node
sudo cat /var/lib/rancher/k3s/server/node-token

# On each agent node — replace SERVER_IP and TOKEN
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

---

## Fedora-Specific Troubleshooting

```bash
# If pods fail to start, check SELinux denials
sudo ausearch -m avc -ts recent | audit2why

# Temporarily set SELinux to permissive for testing (not for production)
sudo setenforce 0
```

---

## Best Practices

- Always install the `k3s-selinux` package before K3s on SELinux-enabled Fedora systems.
- Use `cgroups v2` (default on Fedora 31+) — K3s fully supports it.
- Configure `firewall-cmd --zone=trusted --add-interface=cni0` to allow pod-to-pod communication.
