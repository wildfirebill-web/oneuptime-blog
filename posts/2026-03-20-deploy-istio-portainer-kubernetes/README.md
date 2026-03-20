# How to Deploy Istio via Portainer on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Istio, Service Mesh, DevOps

Description: Learn how to deploy Istio service mesh on Kubernetes using Portainer's Helm and stack management features.

## Introduction

Istio is the leading open-source service mesh that provides traffic management, observability, and security capabilities for microservices running on Kubernetes. Deploying Istio via Portainer simplifies the process by offering a graphical interface to manage Helm charts and Kubernetes manifests.

## Prerequisites

- Portainer Business Edition or CE installed on a Kubernetes cluster
- kubectl access to your cluster
- At least 4 CPU cores and 8 GB RAM available in your cluster
- Portainer connected to your Kubernetes environment

## Step 1: Add the Istio Helm Repository

Navigate to your Kubernetes environment in Portainer:

1. Go to **Kubernetes** > **Helm**
2. Click **Add Repository**
3. Enter the repository details:

```text
Name: istio
URL: https://istio-release.storage.googleapis.com/charts
```

## Step 2: Create the Istio System Namespace

Before installing Istio, create dedicated namespaces:

```yaml
# istio-namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: istio-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: istio-ingress
```

Apply this via Portainer's **Kubernetes** > **Namespaces** > **Add with manifest**.

## Step 3: Deploy Istio Base via Portainer Helm

1. Go to **Kubernetes** > **Helm** > **Charts**
2. Search for `base` in the Istio repository
3. Click **Install**
4. Set namespace to `istio-system`
5. Use these values:

```yaml
# istio-base-values.yaml
defaultRevision: default
```

## Step 4: Deploy Istiod (Control Plane)

1. Find the `istiod` chart
2. Install with namespace `istio-system`
3. Configure values:

```yaml
# istiod-values.yaml
pilot:
  resources:
    requests:
      cpu: 500m
      memory: 2048Mi
  autoscaleEnabled: true
  autoscaleMin: 1
  autoscaleMax: 5

global:
  proxy:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 2000m
        memory: 1024Mi
  tracer:
    zipkin:
      address: ""

meshConfig:
  accessLogFile: /dev/stdout
  enableTracing: true
```

## Step 5: Deploy Istio Ingress Gateway

Deploy the ingress gateway using a Portainer Stack:

```yaml
# istio-ingress-stack.yaml
# Deploy via Portainer Stacks > Add Stack > Kubernetes
apiVersion: v1
kind: Namespace
metadata:
  name: istio-ingress
  labels:
    istio-injection: enabled
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: istio-ingressgateway
  namespace: istio-ingress
spec:
  repo: https://istio-release.storage.googleapis.com/charts
  chart: gateway
  version: 1.20.0
  targetNamespace: istio-ingress
  valuesContent: |-
    service:
      type: LoadBalancer
      ports:
        - port: 15021
          targetPort: 15021
          name: status-port
        - port: 80
          targetPort: 8080
          name: http2
        - port: 443
          targetPort: 8443
          name: https
```

## Step 6: Enable Sidecar Injection

Enable automatic sidecar injection for your application namespaces:

```bash
# Label namespace for sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify the label
kubectl get namespace default --show-labels
```

Or apply via Portainer's namespace editor:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    istio-injection: "enabled"
```

## Step 7: Verify the Istio Installation

Check the installation status through Portainer:

1. Go to **Kubernetes** > **Workloads** > **Pods**
2. Filter by namespace `istio-system`
3. Verify all pods are in **Running** state

Or check via kubectl:

```bash
# Check Istio pods
kubectl get pods -n istio-system

# Check Istio services
kubectl get svc -n istio-system

# Verify istioctl
istioctl verify-install
```

## Step 8: Deploy a Sample Application

Test the service mesh with a sample application stack:

```yaml
# sample-bookinfo-stack.yaml
# Deploy as a Portainer Stack in namespace: default (with istio-injection enabled)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-details
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: details-v1
  namespace: default
  labels:
    app: details
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: details
      version: v1
  template:
    metadata:
      labels:
        app: details
        version: v1
    spec:
      serviceAccountName: bookinfo-details
      containers:
      - name: details
        image: docker.io/istio/examples-bookinfo-details-v1:1.18.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
```

## Monitoring Istio with Portainer

Deploy the Istio observability stack (Kiali, Prometheus, Grafana) as an additional Portainer stack:

```bash
# Install observability addons
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
```

These can be applied via Portainer's **Kubernetes** > **Manifests** section.

## Conclusion

Deploying Istio via Portainer provides a streamlined experience for managing your service mesh infrastructure. The combination of Portainer's visual interface with Istio's powerful service mesh capabilities gives platform teams the ability to manage traffic, enforce security policies, and gain observability across their microservices - all through a unified control plane. By using Portainer's Helm integration and stack management, you can version-control and standardize Istio deployments across multiple Kubernetes clusters.
