# How to Troubleshoot Node Registration Issues in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Nodes

Description: Diagnose and resolve node registration failures in Rancher, including token mismatches, network connectivity, and kubelet configuration issues.

## Introduction

When adding nodes to a Rancher-managed cluster, the node must successfully execute the registration command, run the node agent, and establish communication with both the Rancher server and the cluster's control plane. Failures at any of these steps prevent the node from joining. This guide walks through each failure point.

## Understanding the Node Registration Flow

1. Admin runs the registration `kubectl apply` or `curl | bash` command on the node.
2. The node agent contacts Rancher server to download cluster configuration.
3. The Kubernetes node agent (kubelet) starts and registers with the API server.
4. `cattle-node-agent` starts and connects to `cattle-cluster-agent`.

## Step 1: Verify the Registration Command

```bash
# In Rancher UI: Cluster → Registration → Node Command
# Copy the command — it contains an embedded token tied to the cluster

# The command looks like:
sudo docker run -d --privileged --restart=unless-stopped \
  --net=host -v /etc/kubernetes:/etc/kubernetes \
  -v /var/run:/var/run \
  rancher/rancher-agent:v2.x.y \
  --server https://rancher.example.com \
  --token <registration-token> \
  --worker   # or --controlplane --etcd

# Verify the token hasn't expired
curl -sk -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://<rancher-url>/v3/clusterregistrationtokens?clusterId=<cluster-id>" \
  | jq '.data[].token'
```

## Step 2: Check Node Agent Logs

```bash
# For RKE2 agent nodes
sudo journalctl -u rke2-agent -f --no-pager --lines=200

# For Docker-based registration
docker logs <rancher-agent-container-id> -f --tail=200

# For K3s agent nodes
sudo journalctl -u k3s-agent -f --no-pager --lines=200
```

Common error messages:

| Error | Cause |
|---|---|
| `Failed to connect to proxy` | Node can't reach Rancher server |
| `x509: certificate signed by unknown authority` | CA cert not trusted |
| `token is invalid` | Registration token expired or wrong cluster |
| `node already exists` | Node name conflict (duplicate hostname) |

## Step 3: Test Network Connectivity from the Node

```bash
# The node must reach the Rancher server on port 443
curl -vk https://<rancher-url>/ping
# Expected: "pong"

# The node must also reach the cluster's API server (for RKE2)
# Find the API server endpoint
kubectl get endpoint kubernetes -n default
curl -vk https://<api-server-ip>:6443/readyz

# Check if required ports are blocked by iptables or firewalld
sudo iptables -L INPUT -n -v | grep DROP
sudo firewall-cmd --list-all  # if using firewalld
```

## Step 4: Check for Hostname/Node Name Conflicts

```bash
# Kubernetes node names must be unique
kubectl get nodes
# If a node with the same hostname already exists, the new node can't register

# On the new node, check hostname
hostname

# If there's a duplicate, either:
# 1. Change the hostname on the new node
sudo hostnamectl set-hostname new-node-hostname

# 2. Delete the old node entry from Kubernetes
kubectl delete node <conflicting-node-name>
```

## Step 5: Check for Swap and Firewall Issues

```bash
# Kubernetes (pre-v1.22) requires swap to be disabled
sudo swapon --show
# If swap is active:
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Check SELinux / AppArmor status (may block container runtimes)
sestatus        # SELinux
sudo aa-status  # AppArmor

# For RKE2, br_netfilter and overlay modules must be loaded
sudo modprobe br_netfilter overlay
lsmod | grep -E "br_netfilter|overlay"

# Ensure sysctl settings are correct
sudo sysctl net.bridge.bridge-nf-call-iptables
# Expected: net.bridge.bridge-nf-call-iptables = 1
```

## Step 6: Debug the Kubelet Directly

```bash
# For RKE2, check kubelet logs
sudo /var/lib/rancher/rke2/bin/kubectl \
  --kubeconfig /etc/rancher/rke2/rke2.yaml \
  get nodes

# Kubelet systemd service
sudo journalctl -u kubelet -f --no-pager --lines=200

# Check kubelet configuration
sudo cat /var/lib/rancher/rke2/agent/kubelet.kubeconfig
```

## Step 7: Manually Clean Up a Failed Node

```bash
# Remove the node from Kubernetes
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>

# On the node itself, clean up RKE2 state
sudo /usr/local/bin/rke2-uninstall.sh   # For RKE2
# OR
sudo /usr/local/bin/k3s-uninstall.sh    # For K3s

# Re-run the registration command from Rancher
```

## Conclusion

Node registration issues in Rancher are typically caused by network connectivity problems, token expiration, hostname conflicts, or missing kernel modules. By methodically checking each layer — network, logs, hostname, and system prerequisites — you can quickly identify and resolve the issue and successfully join the node to your cluster.
