# How to Configure K3s with Embedded HA (etcd)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Rancher, etcd, High Availability, HA Cluster

Description: Learn how to set up a highly available K3s cluster using embedded etcd for a fully self-contained HA deployment without external databases.

## Introduction

K3s supports embedded etcd for high availability, allowing you to run multiple server nodes without an external database. This is the simplest HA approach for K3s — all you need are 3 server nodes and a load balancer. The embedded etcd forms a quorum, and the cluster can tolerate one server node failure at a time.

## Why Embedded etcd?

- No external database to manage or pay for
- etcd is the proven, purpose-built Kubernetes datastore
- Supports full cluster recovery from etcd snapshots
- Simpler architecture than managing an external DB cluster

## Requirements for HA

- Odd number of server nodes (3 or 5) for etcd quorum
- Low-latency network between server nodes (< 10ms recommended)
- A load balancer in front of server nodes (HAProxy, NGINX, or cloud LB)

## Etcd Quorum Reference

| Server Nodes | Fault Tolerance | Notes |
|---|---|---|
| 1 | 0 | No HA; single point of failure |
| 3 | 1 | Can lose 1 server |
| 5 | 2 | Can lose 2 servers |

## Step 1: Prepare All Three Server Nodes

On each server node, perform system preparation:

```bash
# On all 3 server nodes
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable kernel modules
sudo modprobe br_netfilter overlay
echo "br_netfilter" | sudo tee /etc/modules-load.d/k3s.conf

# Set sysctl settings
sudo tee /etc/sysctl.d/k3s.conf > /dev/null <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

## Step 2: Bootstrap the First Server

The first server node initializes the etcd cluster with `--cluster-init`:

```bash
# On the FIRST server node (192.168.1.101)
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
# Initialize a new embedded etcd cluster
cluster-init: true

# Shared cluster token (same on all server nodes)
token: "HAClusterToken"

# TLS SANs - include the load balancer VIP
tls-san:
  - 192.168.1.100    # Load balancer VIP
  - 192.168.1.101    # This server node
  - k3s-ha.example.com

# Kubelet configuration
kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=300m,memory=512Mi"
  - "system-reserved=cpu=200m,memory=256Mi"

# etcd configuration
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 5
EOF

# Install K3s
curl -sfL https://get.k3s.io | sudo sh -

# Verify the first server is running
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

## Step 3: Join the Second and Third Server Nodes

```bash
# On the SECOND server node (192.168.1.102)
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
# Join the existing cluster (no cluster-init)
server: "https://192.168.1.101:6443"
token: "HAClusterToken"

tls-san:
  - 192.168.1.100
  - 192.168.1.102
  - k3s-ha.example.com

kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=300m,memory=512Mi"
  - "system-reserved=cpu=200m,memory=256Mi"

etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 5
EOF

curl -sfL https://get.k3s.io | sudo sh -
```

```bash
# On the THIRD server node (192.168.1.103)
sudo mkdir -p /etc/rancher/k3s

sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
server: "https://192.168.1.101:6443"
token: "HAClusterToken"

tls-san:
  - 192.168.1.100
  - 192.168.1.103
  - k3s-ha.example.com

kubelet-arg:
  - "max-pods=110"
  - "kube-reserved=cpu=300m,memory=512Mi"
  - "system-reserved=cpu=200m,memory=256Mi"

etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 5
EOF

curl -sfL https://get.k3s.io | sudo sh -
```

## Step 4: Set Up the Load Balancer

Configure HAProxy in front of the three server nodes:

```bash
# Install HAProxy
sudo apt-get install -y haproxy

# Configure HAProxy
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOF
global
    log /dev/log local0
    maxconn 4096
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# K3s API Server load balancing
frontend k3s_api
    bind *:6443
    default_backend k3s_servers

backend k3s_servers
    balance roundrobin
    option tcp-check
    server k3s-server-01 192.168.1.101:6443 check
    server k3s-server-02 192.168.1.102:6443 check
    server k3s-server-03 192.168.1.103:6443 check

# K3s Supervisor API (for agent registration)
frontend k3s_supervisor
    bind *:9345
    default_backend k3s_supervisors

backend k3s_supervisors
    balance roundrobin
    option tcp-check
    server k3s-server-01 192.168.1.101:9345 check
    server k3s-server-02 192.168.1.102:9345 check
    server k3s-server-03 192.168.1.103:9345 check
EOF

sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

## Step 5: Add Agent Nodes

```bash
# On each agent node, connect through the load balancer
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<EOF
server: "https://192.168.1.100:6443"
token: "HAClusterToken"
node-label:
  - "tier=worker"
EOF

curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="agent" \
    sudo sh -
```

## Step 6: Verify the HA Cluster

```bash
# Get kubeconfig from the first server
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Update the server URL to use the load balancer
sed -i 's/127.0.0.1/192.168.1.100/' ~/.kube/config

# Check all nodes
kubectl get nodes -o wide

# Verify etcd cluster health
sudo k3s etcd-snapshot list

# Check etcd member count (should be 3)
sudo k3s kubectl -n kube-system exec etcd-k3s-server-01 -- \
    etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
    --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
    --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
    member list
```

## Step 7: Test HA Failover

```bash
# Simulate a server failure
ssh 192.168.1.101 "sudo systemctl stop k3s"

# Verify the cluster is still operational
kubectl get nodes
kubectl get pods --all-namespaces

# Check a deployment continues to run
kubectl create deployment ha-test --image=nginx --replicas=3
kubectl get pods -o wide

# Restore the failed server
ssh 192.168.1.101 "sudo systemctl start k3s"

# Confirm it rejoined
kubectl get nodes
```

## Managing etcd Snapshots

```bash
# List existing snapshots
sudo k3s etcd-snapshot list

# Create a manual snapshot
sudo k3s etcd-snapshot save --name ha-cluster-backup

# Restore from a snapshot (cluster must be stopped)
sudo k3s server --cluster-reset --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/ha-cluster-backup
```

## Conclusion

K3s with embedded etcd provides a self-contained, production-ready HA Kubernetes cluster that requires no external database infrastructure. By using 3 server nodes with `cluster-init: true` on the first node and pointing subsequent nodes at the first server, you get a fully redundant control plane. Combine this with a load balancer for the API endpoint and automated etcd snapshots for disaster recovery, and you have a robust HA cluster that's simpler to operate than traditional Kubernetes distributions.
