# How to Toggle System Resource Visibility in Portainer for Kubernetes (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, System Resources, UI, Cluster Management

Description: Learn how to show or hide Kubernetes system resources and namespaces in Portainer to keep the interface clean or do deep cluster inspection.

## What Are System Resources?

Kubernetes system resources are workloads and configurations that manage the cluster itself, running in:

- `kube-system` - Core components (kube-dns, kube-proxy, metrics-server, CNI plugins)
- `kube-public` - Public cluster information
- `portainer` - The Portainer agent itself

By default, Portainer hides these to keep the UI focused on user workloads.

## Toggling System Resource Visibility

In Portainer, within your Kubernetes environment:

1. Navigate to **Applications**.
2. Look for the **Show system resources** toggle (usually a checkbox or toggle switch at the top of the list).
3. Enable it to show system namespaces and their workloads.

## What Becomes Visible

After enabling system resources:

- System namespaces (`kube-system`, `kube-public`, `portainer`) appear in namespace lists.
- System pods (CoreDNS, kube-proxy, network plugin pods) appear in the applications list.
- System ConfigMaps and Secrets become visible.

## Viewing System Namespaces via CLI

```bash
# List all namespaces including system namespaces

kubectl get namespaces

# View pods in the kube-system namespace
kubectl get pods --namespace=kube-system

# Check DNS pods specifically
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check all system components status
kubectl get componentstatuses

# View system resource usage
kubectl top pods --namespace=kube-system
```

## Inspecting kube-system Components

```bash
# View CoreDNS configuration
kubectl get configmap coredns --namespace=kube-system -o yaml

# Check kube-proxy daemonset
kubectl get daemonset kube-proxy --namespace=kube-system

# View all system ConfigMaps
kubectl get configmaps --namespace=kube-system
```

## When to Enable System Resource Visibility

**Enable when:**
- Debugging DNS resolution issues (CoreDNS)
- Troubleshooting network connectivity (CNI pods)
- Checking metrics-server or other cluster add-ons
- Auditing what's running in the cluster
- Investigating resource pressure in system namespaces

**Keep disabled when:**
- Normal day-to-day application management
- Sharing the Portainer view with developers who don't need system access
- Reducing UI clutter in large clusters

## Important: Don't Modify System Resources

Unless you know exactly what you're doing, avoid editing or deleting system resources from Portainer. Modifying kube-system workloads can break DNS, networking, or cluster control plane functions.

## Conclusion

Portainer's system resource visibility toggle is a simple but useful feature for cluster operators. Keep it hidden for daily app management, and enable it when you need to inspect or troubleshoot cluster-level components.
