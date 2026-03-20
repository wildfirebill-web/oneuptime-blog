# How to Configure RKE2 Server Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Server Configuration, SUSE Rancher, Cluster Setup, Node Management

Description: Learn how to configure RKE2 server nodes including the configuration file options, token setup, API server flags, and cluster initialization for a production-ready control plane.

---

RKE2 (Rancher Kubernetes Engine 2) is a fully conformant Kubernetes distribution focused on security and compliance. Server nodes run the Kubernetes control plane components: kube-apiserver, kube-controller-manager, kube-scheduler, and etcd.

---

## Step 1: Install RKE2 Server

```bash
# Download and run the RKE2 installer
curl -sfL https://get.rke2.io | sh -

# Enable and start the RKE2 server service
systemctl enable rke2-server.service
```

---

## Step 2: Create the Server Configuration File

RKE2 reads its configuration from `/etc/rancher/rke2/config.yaml`. Create this file before starting the service:

```yaml
# /etc/rancher/rke2/config.yaml

# Shared cluster secret — all nodes must use the same token
token: my-super-secret-cluster-token

# TLS SANs added to the API server certificate
tls-san:
  - "rancher.example.com"
  - "192.168.1.10"
  - "api.my-cluster.internal"

# The CIDR for pods
cluster-cidr: 10.42.0.0/16

# The CIDR for services
service-cidr: 10.43.0.0/16

# DNS cluster IP (must be in service-cidr)
cluster-dns: 10.43.0.10

# CNI plugin to use (cilium, calico, canal)
cni: cilium

# Disable components not needed in air-gapped environments
disable:
  - rke2-ingress-nginx   # disable default ingress if using a custom one

# Write kubeconfig with these permissions
write-kubeconfig-mode: "0640"

# Additional kube-apiserver flags
kube-apiserver-arg:
  - "audit-log-path=/var/log/kube-audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
```

---

## Step 3: Start the First Server Node

```bash
# Start the RKE2 server (initializes the cluster on the first run)
systemctl start rke2-server.service

# Follow the startup logs
journalctl -u rke2-server -f
```

After the service starts, the kubeconfig is available:

```bash
# Add RKE2 tools to PATH
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

# Verify the node is ready
kubectl get nodes
```

---

## Step 4: Retrieve the Join Token

Additional server nodes and agent nodes need the token to join:

```bash
# Print the cluster join token
cat /var/lib/rancher/rke2/server/node-token
```

---

## Step 5: Configure Additional Control Plane Nodes

On each additional server node, set `server` to the first node's address:

```yaml
# /etc/rancher/rke2/config.yaml (on node 2 and 3)
server: https://192.168.1.10:9345
token: my-super-secret-cluster-token
tls-san:
  - "rancher.example.com"
  - "192.168.1.10"
```

```bash
systemctl enable --now rke2-server.service
```

---

## Useful Paths

| Path | Purpose |
|---|---|
| `/etc/rancher/rke2/config.yaml` | Server configuration |
| `/var/lib/rancher/rke2/server/node-token` | Join token |
| `/etc/rancher/rke2/rke2.yaml` | Admin kubeconfig |
| `/var/lib/rancher/rke2/agent/logs/kubelet.log` | Kubelet logs |

---

## Best Practices

- Always set `tls-san` to include your load balancer DNS name before starting the first server.
- Store the token in a secrets manager, not plaintext in configuration management.
- Run 3 or 5 server nodes for HA — never 2 or 4 (etcd requires an odd quorum).
