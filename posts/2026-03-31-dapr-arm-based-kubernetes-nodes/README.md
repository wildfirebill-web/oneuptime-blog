# How to Use Dapr with ARM-Based Kubernetes Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, ARM, Kubernetes, Multi-Architecture, Cost Optimization

Description: Deploy Dapr on ARM-based Kubernetes nodes (AWS Graviton, Azure Ampere) to reduce compute costs while maintaining full Dapr functionality across all building blocks.

---

## Dapr on ARM Architecture

Dapr publishes multi-architecture container images that support both AMD64 (x86_64) and ARM64. Running on ARM-based nodes like AWS Graviton or Azure Ampere can reduce compute costs by 20-40% for the same workload.

## Verifying ARM64 Support

Check that the Dapr images support ARM64:

```bash
# Verify Dapr runtime image supports ARM64
docker buildx imagetools inspect \
  daprio/daprd:1.13.0 | grep arm64

# Expected output:
# linux/arm64    sha256:...
```

All official Dapr images from `daprio/` on Docker Hub are multi-arch.

## Creating ARM Node Groups on AWS EKS

```bash
# Create ARM node group using Graviton processors
eksctl create nodegroup \
  --cluster my-cluster \
  --name arm-nodegroup \
  --node-type m7g.xlarge \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 5 \
  --amd64                  # Flag ensures AMD64 system pods run elsewhere
```

Label ARM nodes:

```bash
kubectl label nodes -l kubernetes.io/arch=arm64 \
  workload-type=compute-optimized
```

## Deploying Dapr Control Plane on ARM

Install Dapr with node affinity for ARM:

```bash
helm install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --set global.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution=true
```

Or pin to ARM nodes explicitly in the Helm values:

```yaml
# dapr-values-arm.yaml
dapr_operator:
  nodeSelector:
    kubernetes.io/arch: arm64

dapr_sentry:
  nodeSelector:
    kubernetes.io/arch: arm64

dapr_placement:
  nodeSelector:
    kubernetes.io/arch: arm64
```

## Deploying Dapr-Enabled Applications on ARM

Your application images must also be multi-arch. Build with Docker Buildx:

```bash
# Set up multi-arch builder
docker buildx create --use --name multiarch-builder
docker buildx inspect --bootstrap

# Build and push multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag myregistry/order-service:latest \
  --push .
```

Deploy to ARM nodes:

```yaml
spec:
  nodeSelector:
    kubernetes.io/arch: arm64
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
```

## Validating Dapr Functionality on ARM

Run the Dapr health check from an ARM pod:

```bash
# Check sidecar is healthy
kubectl exec -it order-service-xxx -c daprd -- \
  wget -qO- http://localhost:3500/v1.0/healthz

# Verify ARM architecture in the sidecar
kubectl exec -it order-service-xxx -c daprd -- uname -m
# Expected: aarch64
```

## Performance Comparison

Measure Dapr operation throughput on ARM vs AMD64:

```bash
# Run hey load test on ARM node
hey -n 10000 -c 50 \
  http://arm-service:3500/v1.0/state/statestore/test-key

# Run same test on AMD64 node
hey -n 10000 -c 50 \
  http://amd64-service:3500/v1.0/state/statestore/test-key
```

ARM Graviton typically shows 5-15% lower throughput but 25-40% lower cost per operation.

## Mixed Architecture Clusters

In mixed clusters (ARM and AMD64 nodes), use node affinity in your workloads to control placement:

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
            - key: kubernetes.io/arch
              operator: In
              values: [arm64]
```

## Summary

Dapr fully supports ARM64 through its multi-architecture container images and has been validated on AWS Graviton and Azure Ampere nodes. Building application images as multi-arch with Docker Buildx, setting node selectors for ARM node groups, and validating sidecar health post-deployment enables 25-40% compute cost reductions while maintaining full Dapr building block functionality.
