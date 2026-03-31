# How to Use Dapr Shared Mode (Per-Node Deployment) on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Shared Mode, DaemonSet, Performance

Description: Configure Dapr in shared sidecar mode where a single Dapr process runs per node as a DaemonSet, reducing resource overhead in high-density clusters.

---

## What Is Dapr Shared Mode?

By default, Dapr injects a sidecar container into every pod. In high-density environments with many small pods per node, this creates significant memory and CPU overhead. Dapr Shared Mode (also called Dapr Shared or per-node deployment) runs a single Dapr process per node as a DaemonSet, which all pods on that node share.

## When to Use Shared Mode

Use shared mode when:
- You have many small pods per node (50+)
- Sidecar memory overhead is a concern
- You are running Dapr on resource-constrained nodes

## Installing Dapr Shared

Dapr Shared is installed as a separate Helm chart alongside the main Dapr installation:

```bash
# Install the main Dapr control plane first
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update
helm install dapr dapr/dapr --namespace dapr-system --wait

# Install dapr-shared as a DaemonSet
helm install dapr-shared dapr/dapr-shared \
  --namespace dapr-system \
  --set shared.appId="shared-dapr" \
  --set shared.remoteURL="dapr-api.dapr-system.svc.cluster.local" \
  --set shared.remotePort="80" \
  --wait
```

## Annotating Applications for Shared Mode

Instead of the standard `dapr.io/enabled: "true"` annotation, use the shared mode annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "my-service"
        dapr.io/app-port: "8080"
        dapr.io/sidecar-listen-addresses: "0.0.0.0"
      labels:
        app: my-service
    spec:
      containers:
      - name: my-service
        image: myregistry/my-service:latest
        ports:
        - containerPort: 8080
```

## Verifying the DaemonSet

```bash
# Check the DaemonSet is running on all nodes
kubectl get daemonset -n dapr-system
# NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# dapr-shared   3         3         3       3            3

# Verify each node has a shared dapr pod
kubectl get pods -n dapr-system -l app=dapr-shared -o wide
```

## Resource Savings Calculation

```bash
# With standard sidecars: 50 pods x 50Mi memory = 2500Mi per node
# With shared mode: 1 DaemonSet pod x 128Mi = 128Mi per node
# Savings: ~95% memory reduction for high-density workloads

# Check actual resource usage
kubectl top pods -n dapr-system -l app=dapr-shared
```

## Limitations of Shared Mode

Shared mode does not support all Dapr features:
- Actor placement is not supported in shared mode
- Each pod still needs its own app-id annotation
- Isolation between apps sharing the same node process is reduced

## Summary

Dapr Shared Mode deploys a single Dapr DaemonSet process per node instead of per-pod sidecars, dramatically reducing resource consumption in high-density clusters. It is best suited for stateless microservices that do not use Dapr Actors and run many instances per node.
