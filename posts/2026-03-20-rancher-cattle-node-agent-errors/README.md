# How to Troubleshoot cattle-node-agent Errors in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Agent, Node

Description: Learn how to diagnose and fix cattle-node-agent errors in Rancher, including container runtime issues, volume mount failures, and network problems.

## Introduction

The `cattle-node-agent` runs as a DaemonSet on every node in a Rancher-managed cluster. It handles node-level operations such as node provisioning, OS-level configuration, and maintaining the connection back to the cluster agent. When node agents fail, affected nodes may show as "Not Ready" or unavailable for workloads.

## Architecture

```text
cattle-cluster-agent (one per cluster, in cattle-system)
       ↓ manages
cattle-node-agent (DaemonSet, one pod per node)
```

The node agent runs with `hostNetwork: true` and `privileged: true` to access host resources.

## Step 1: Check Node Agent Status

```bash
# Check the DaemonSet status

kubectl get daemonset -n cattle-system cattle-node-agent

# Check pods - look for nodes where the agent is not Running
kubectl get pods -n cattle-system -l app=cattle-agent -o wide

# Find nodes where the agent is failing
kubectl get pods -n cattle-system -l app=cattle-agent \
  | grep -v "Running\|Completed"

# Get logs from a specific node's agent
NODE_AGENT=$(kubectl get pod -n cattle-system -l app=cattle-agent \
  --field-selector spec.nodeName=<node-name> -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n cattle-system ${NODE_AGENT} --tail=200
```

## Step 2: Diagnose Image Pull Failures

```bash
# If the node agent pod shows ImagePullBackOff
kubectl describe pod -n cattle-system ${NODE_AGENT} | grep -A5 "Events:"

# For air-gapped clusters, mirror the agent image
# On a node with internet access:
docker pull rancher/rancher-agent:v2.9.0
docker save rancher/rancher-agent:v2.9.0 | gzip > rancher-agent.tar.gz

# Transfer to the air-gapped node and load
scp rancher-agent.tar.gz user@<node-ip>:~
ssh user@<node-ip> 'sudo ctr images import rancher-agent.tar.gz'

# For containerd (used by RKE2/K3s):
ssh user@<node-ip> 'sudo /var/lib/rancher/rke2/bin/ctr \
  --address /run/k3s/containerd/containerd.sock \
  images import rancher-agent.tar.gz'
```

## Step 3: Debug Volume Mount Issues

The node agent mounts host paths for container runtime sockets:

```bash
# Check what volumes the node agent mounts
kubectl get daemonset -n cattle-system cattle-node-agent -o json \
  | jq '.spec.template.spec.volumes[]'

# Common required paths:
# /var/run/docker.sock   - Docker (older clusters)
# /run/containerd/containerd.sock - Containerd
# /run/k3s/containerd/containerd.sock - K3s/RKE2

# On the node, verify the socket exists
ls -la /run/containerd/containerd.sock
ls -la /run/k3s/containerd/containerd.sock

# If the wrong socket path is configured, update the DaemonSet
kubectl edit daemonset -n cattle-system cattle-node-agent
```

## Step 4: Check Host Network Issues

The node agent uses `hostNetwork: true` and binds to the node's IP:

```bash
# Verify the node's IP is accessible
kubectl get node <node-name> -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'

# Check if another process is using the agent's port
ssh user@<node-ip> 'sudo ss -tlnp | grep 10250'

# Check for iptables rules that might block the agent
ssh user@<node-ip> 'sudo iptables -L INPUT -n -v | grep DROP'
```

## Step 5: Check Disk Pressure and System Resources

```bash
# Node agents fail if the node has disk pressure
kubectl describe node <node-name> | grep -A10 "Conditions:"

# Check disk usage on the node
ssh user@<node-ip> 'df -h'

# Check available inodes (often the real culprit)
ssh user@<node-ip> 'df -i'

# Free up disk space by removing unused container images
ssh user@<node-ip> 'sudo crictl rmi --prune'
# OR for Docker nodes:
ssh user@<node-ip> 'sudo docker image prune -af'
```

## Step 6: Fix Nodes Stuck in NotReady

When a node is stuck in `NotReady` due to agent issues:

```bash
# Check kubelet status on the node
ssh user@<node-ip> 'sudo systemctl status kubelet'
ssh user@<node-ip> 'sudo journalctl -u kubelet -n 100 --no-pager'

# Restart the kubelet
ssh user@<node-ip> 'sudo systemctl restart kubelet'

# For RKE2:
ssh user@<node-ip> 'sudo systemctl restart rke2-agent'

# Force the DaemonSet to recreate the pod on the node
kubectl delete pod -n cattle-system ${NODE_AGENT}
```

## Step 7: Verify After Recovery

```bash
# Check node status
kubectl get nodes

# Verify all node agent pods are running
kubectl get pods -n cattle-system -l app=cattle-agent -o wide

# Check that the node is reporting to the cluster correctly
kubectl describe node <node-name> | grep -E "Conditions:|Ready:"
```

## Conclusion

`cattle-node-agent` failures are usually caused by image pull issues in air-gapped environments, missing container runtime sockets, disk pressure on nodes, or network connectivity problems. Monitoring the DaemonSet rollout status and individual pod health across all nodes is essential for maintaining cluster stability. Proactive node maintenance - clearing disk space and keeping container images up to date - prevents most node agent failures.
