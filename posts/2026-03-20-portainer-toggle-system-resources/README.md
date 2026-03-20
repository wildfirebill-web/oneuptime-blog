# How to Toggle System Resource Visibility in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, System Resources, Namespaces, DevOps

Description: Learn how to show or hide Kubernetes system resources in Portainer to keep your view clean while still being able to access system-level workloads when needed.

## Introduction

Kubernetes runs system workloads in namespaces like `kube-system`, `kube-public`, and `portainer`. These system resources are hidden by default in Portainer to keep the interface clean and prevent accidental modification. However, administrators sometimes need to inspect or troubleshoot system components. This guide explains how to toggle system resource visibility in Portainer.

## Prerequisites

- Portainer with Kubernetes environment connected
- Admin access to Portainer

## Understanding System Namespaces

Kubernetes uses several reserved namespaces for system components:

```text
kube-system       - Core Kubernetes components (kube-dns, kube-proxy, metrics-server)
kube-public       - Publicly readable cluster information
kube-node-lease   - Node heartbeat lease objects
portainer         - Portainer's own agent and namespace
ingress-nginx     - Nginx Ingress Controller (if installed)
cert-manager      - Certificate manager (if installed)
monitoring        - Prometheus/Grafana stack (if installed)
```

## Step 1: Show System Namespaces in Portainer

By default, Portainer hides system namespaces from the namespace dropdown. To show them:

1. Select your Kubernetes environment
2. In the namespace dropdown (top of screen), look for a toggle or checkbox
3. Enable **Show system namespaces**
4. System namespaces (kube-system, kube-public, kube-node-lease) now appear in the list

Alternatively, access system resources directly:
1. Click **Namespaces** in the sidebar
2. Enable **Show system resources** toggle at the top of the namespace list

## Step 2: Access System Resources via kubectl

When troubleshooting, access system namespaces directly via kubectl:

```bash
# View all system pods

kubectl get pods -n kube-system

# Common system components:
# NAME                                    READY   STATUS    RESTARTS
# coredns-xxx                             1/1     Running   0
# etcd-master                             1/1     Running   0
# kube-apiserver-master                   1/1     Running   0
# kube-controller-manager-master          1/1     Running   0
# kube-proxy-xxx                          1/1     Running   0
# kube-scheduler-master                   1/1     Running   0
# metrics-server-xxx                      1/1     Running   0

# View deployments in kube-system
kubectl get deployments -n kube-system

# Check CoreDNS configuration
kubectl describe configmap coredns -n kube-system

# View system events
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

## Step 3: Configure Portainer to Always Show System Resources

For admin users who regularly need system resource access:

1. Go to **Settings** → **Kubernetes** settings for the environment
2. Find the **System resource visibility** option
3. Choose a default visibility setting:
   - **Hidden** (default) - System namespaces not shown in namespace picker
   - **Visible** - Always show system namespaces

This setting persists for the environment and applies to all users with admin access.

## Step 4: View kube-system Workloads in Portainer

Once system namespaces are visible:

1. Select `kube-system` from the namespace dropdown
2. Navigate to **Applications** to see system workloads:

```text
kube-dns / coredns       - Cluster DNS resolution
kube-proxy               - Network rules on each node
metrics-server           - Resource metrics for HPA
calico / flannel / cilium - CNI network plugin
```

3. You can view logs and resource usage, but exercise caution when modifying system resources

## Step 5: Inspect System Component Logs

System component logs are critical for cluster troubleshooting:

```bash
# CoreDNS logs (DNS resolution issues)
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# kube-proxy logs (network issues)
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# metrics-server logs (HPA issues)
kubectl logs -n kube-system -l k8s-app=metrics-server --tail=50

# API server logs (on control plane node)
kubectl logs -n kube-system kube-apiserver-<node-name> --tail=100

# View events across all namespaces including system
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

## Step 6: Hide System Resources for Non-Admin Users

In Portainer BE, you can enforce visibility restrictions per team:

1. When configuring namespace access for a team, do not grant access to `kube-system`
2. Users in that team will never see system namespaces, regardless of toggle settings
3. Admins retain full visibility

This prevents non-admin users from accidentally viewing or attempting to modify critical system components.

## Step 7: Portainer's Own Namespace

The `portainer` namespace contains the Portainer agent:

```bash
# View Portainer's resources
kubectl get all -n portainer

# Check agent status
kubectl get pods -n portainer

# View agent logs
kubectl logs -n portainer -l app=portainer-agent

# View Portainer server (if deployed in Kubernetes)
kubectl get pods -n portainer -l app=portainer
```

Be careful not to delete resources in the `portainer` namespace as it will disconnect your Portainer instance from the cluster.

## Step 8: Checking System Namespace Health

Monitor system namespaces to detect cluster health issues:

```bash
# Check all system pods are Running
kubectl get pods -n kube-system | grep -v Running
# Should return no results in a healthy cluster

# Check for any pending or failed system pods
kubectl get pods -A | grep -E "Pending|Failed|Error|CrashLoop"

# Check node conditions
kubectl describe nodes | grep -A 5 "Conditions:"

# View recent system events with warnings
kubectl get events -n kube-system --field-selector type=Warning
```

## Conclusion

System resource visibility in Portainer is hidden by default to protect critical cluster components and keep the interface clean for application developers. Administrators can toggle visibility when troubleshooting cluster-level issues. Use this feature judiciously - inspect system resources when needed, but avoid making changes to system namespaces unless you understand the full impact. For Portainer BE deployments, access control ensures non-admin users never accidentally interact with system resources.
