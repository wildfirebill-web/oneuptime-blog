# How to Deploy Knative on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Knative, Serverless, Kubernetes, Event-Driven, Auto-Scaling

Description: Deploy Knative Serving and Eventing on Rancher to enable serverless workloads with scale-to-zero, traffic splitting, and event-driven processing.

## Introduction

Knative extends Kubernetes with serverless capabilities: automatic scale-to-zero, scale-from-zero on demand, traffic splitting for progressive rollouts, and an event-driven architecture through Knative Eventing. It's the most popular serverless framework for Kubernetes.

## Prerequisites

- Rancher cluster with at least 3 worker nodes
- An Ingress controller (Kourier, Istio, or Contour)
- `kubectl` configured for the cluster

## Step 1: Install Knative Serving

```bash
# Install Knative Serving CRDs
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.14.0/serving-crds.yaml

# Install Knative Serving core
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.14.0/serving-core.yaml
```

## Step 2: Install Kourier Networking Layer

```bash
# Install Kourier as the network layer
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.14.0/kourier.yaml

# Configure Knative Serving to use Kourier
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

## Step 3: Configure DNS

```bash
# Configure magic DNS (for development)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.14.0/serving-default-domain.yaml

# Or configure real DNS
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"example.com":""}}'
```

## Step 4: Deploy a Knative Service

```yaml
# hello-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
  namespace: default
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"    # Scale to zero
        autoscaling.knative.dev/max-scale: "10"   # Max 10 replicas
        autoscaling.knative.dev/target: "100"     # Target 100 concurrent requests
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Rancher"
```

```bash
kubectl apply -f hello-service.yaml

# Get the service URL
kubectl get ksvc hello
```

## Step 5: Traffic Splitting

```yaml
# Traffic splitting between revisions
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  traffic:
    - revisionName: hello-v1
      percent: 80    # 80% to stable version
    - revisionName: hello-v2
      percent: 20    # 20% canary release
```

## Step 6: Install Knative Eventing

```bash
# Install Knative Eventing
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.14.0/eventing-crds.yaml
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.14.0/eventing-core.yaml
```

## Conclusion

Knative on Rancher transforms your cluster into a serverless platform. Scale-to-zero reduces resource costs during quiet periods, while the Knative Eventing system enables loosely coupled event-driven architectures. The traffic splitting feature makes progressive deployments straightforward with no additional tooling required.
