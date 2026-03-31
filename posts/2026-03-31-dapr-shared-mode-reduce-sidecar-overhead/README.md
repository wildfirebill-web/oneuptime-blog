# How to Use Dapr Shared Mode to Reduce Sidecar Overhead

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, Performance, Sidecar, Resource Optimization

Description: Learn how Dapr shared mode deploys a single Dapr process per node instead of per pod, reducing memory and CPU overhead in high-density workloads.

---

## What Is Dapr Shared Mode?

By default, Dapr injects a sidecar container into every pod. In clusters with hundreds of pods per node, this multiplies resource consumption significantly. Dapr shared mode (also called "dapr-shared") solves this by running a single Dapr process on each node that all pods on that node share.

This is especially useful for:
- Batch workloads with short-lived pods
- Edge deployments with constrained resources
- Clusters with many small services that rarely use Dapr features

## Enabling Dapr Shared Mode

Install the dapr-shared Helm chart alongside the standard Dapr installation:

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update

helm install dapr-shared dapr/dapr-shared \
  --namespace dapr-system \
  --set shared.appId=my-app \
  --set shared.remoteURL=localhost \
  --set shared.remotePort=3000
```

## Annotating Applications to Use Shared Mode

Instead of the standard Dapr sidecar injection annotation, use the shared mode annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 10
  template:
    metadata:
      annotations:
        dapr.io/enabled: "false"
        dapr.io/app-id: "order-processor"
        dapr.io/shared-mode: "true"
    spec:
      containers:
      - name: order-processor
        image: myregistry/order-processor:latest
        ports:
        - containerPort: 3000
        env:
        - name: DAPR_HTTP_PORT
          value: "3500"
        - name: DAPR_GRPC_PORT
          value: "50001"
```

## Measuring the Resource Savings

Measure memory savings by comparing pod resource usage with and without shared mode:

```bash
# Check resource usage per pod with standard sidecar
kubectl top pods -n production --sort-by=memory | head -20

# After enabling shared mode, compare
kubectl top pods -n production --sort-by=memory | head -20

# Check the shared dapr process on a node
kubectl top pods -n dapr-system | grep dapr-shared
```

In a typical scenario with 50 pods per node, standard sidecar mode uses roughly 50 x 20 MB = 1 GB per node just for Dapr sidecars. Shared mode reduces this to a single ~60 MB process per node.

## Configuring Shared Mode Performance

Tune the shared Dapr process for your workload:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dapr-shared-config
  namespace: dapr-system
data:
  config.yaml: |
    apiVersion: dapr.io/v1alpha1
    kind: Configuration
    metadata:
      name: shared-config
    spec:
      tracing:
        samplingRate: "0.1"
      metric:
        enabled: true
      httpPipeline:
        handlers:
        - name: ratelimit
          type: middleware.http.ratelimit
```

## Limitations of Shared Mode

Shared mode has trade-offs to consider:

- App ID must be consistent across pods using the same shared process
- Not suitable when each pod needs isolated Dapr state or different component access
- mTLS and identity isolation is reduced compared to per-pod sidecar
- Works best for stateless, read-heavy workloads

For compliance-sensitive applications where pod-level isolation is required, the standard sidecar model remains appropriate.

## Summary

Dapr shared mode deploys a single Dapr process per Kubernetes node instead of injecting a sidecar into every pod, reducing memory overhead by up to 90% in high-density clusters. It is configured via the dapr-shared Helm chart and special pod annotations. Evaluate the isolation trade-offs before enabling shared mode in production environments where per-pod security boundaries are required.
