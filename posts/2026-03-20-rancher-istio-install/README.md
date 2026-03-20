# How to Install Istio from Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, Service Mesh, DevOps

Description: A step-by-step guide to installing Istio service mesh on a Kubernetes cluster managed by Rancher.

Istio is a powerful open-source service mesh that provides traffic management, observability, and security features for microservices running on Kubernetes. Rancher makes it straightforward to deploy and manage Istio through its built-in Apps & Marketplace catalog. This guide walks you through the complete installation process.

## Prerequisites

Before installing Istio from Rancher, ensure you have the following in place:

- Rancher v2.6 or later installed and accessible
- A downstream Kubernetes cluster (RKE, RKE2, or managed cloud cluster) with at least 4 vCPUs and 8 GB RAM
- `kubectl` configured to communicate with your cluster
- Cluster admin privileges in Rancher

## Step 1: Enable the Istio Feature Flag

Rancher ships with Istio support available through the Rancher Apps catalog. First, verify that the Istio feature is available in your Rancher instance.

1. Log in to the Rancher UI
2. Navigate to **Cluster Management** and select your target cluster
3. Go to **Apps** → **Charts**
4. Search for **Istio** in the catalog

## Step 2: Configure Namespace and Resource Requirements

Istio components will be installed into the `istio-system` namespace. Rancher creates this automatically during installation.

Recommended resource allocations for a production setup:

| Component | CPU Request | Memory Request |
|---|---|---|
| istiod | 500m | 2Gi |
| istio-ingressgateway | 100m | 128Mi |
| istio-egressgateway | 100m | 128Mi |

## Step 3: Install Istio via Rancher Apps

1. In the Rancher UI, navigate to **Apps** → **Charts** on your target cluster
2. Locate the **Istio** chart and click **Install**
3. Select the namespace `istio-system` (create it if it does not exist)
4. Configure the Helm values as needed

```yaml
# Example values.yaml for Istio installation via Rancher

global:
  # Set the Istio control plane profile
  # Options: default, demo, minimal, remote
  profile: default

pilot:
  # Resource requests for istiod
  resources:
    requests:
      cpu: 500m
      memory: 2048Mi

gateways:
  istio-ingressgateway:
    # Enable the ingress gateway
    enabled: true
    type: LoadBalancer
    resources:
      requests:
        cpu: 100m
        memory: 128Mi

  istio-egressgateway:
    # Enable the egress gateway
    enabled: true
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
```

5. Click **Install** to deploy Istio

## Step 4: Verify the Installation

After the installation completes, verify all Istio pods are running:

```bash
# Check that all Istio components are running
kubectl get pods -n istio-system

# Expected output:
# NAME                                   READY   STATUS    RESTARTS   AGE
# istiod-xxxxxxxxx-xxxxx                 1/1     Running   0          2m
# istio-ingressgateway-xxxxxxxxx-xxxxx   1/1     Running   0          2m
# istio-egressgateway-xxxxxxxxx-xxxxx    1/1     Running   0          2m
```

```bash
# Verify the Istio installation using istioctl
kubectl get services -n istio-system

# Check the Istio ConfigMap
kubectl get configmap istio -n istio-system -o yaml
```

## Step 5: Install istioctl (Optional but Recommended)

The `istioctl` CLI provides additional management capabilities:

```bash
# Download the latest istioctl binary
curl -L https://istio.io/downloadIstio | sh -

# Move to a directory in your PATH
sudo mv istio-*/bin/istioctl /usr/local/bin/

# Verify the installation
istioctl version

# Analyze your cluster's Istio configuration
istioctl analyze
```

## Step 6: Enable Istio Injection for Namespaces

To have Istio automatically inject sidecar proxies into your application pods, label your namespaces:

```bash
# Enable automatic sidecar injection for a namespace
kubectl label namespace default istio-injection=enabled

# Verify the label was applied
kubectl get namespace default --show-labels
```

## Monitoring Istio with OneUptime

After installing Istio, integrating with a monitoring platform like OneUptime helps you track the health of your service mesh. OneUptime can monitor your Istio ingress gateway endpoints and alert you when services become unavailable.

```bash
# Get the external IP of the Istio ingress gateway
kubectl get svc istio-ingressgateway -n istio-system

# Use this IP to configure monitors in OneUptime
```

## Conclusion

Installing Istio from Rancher is a streamlined process thanks to the built-in Apps catalog. Once installed, Istio provides a robust foundation for traffic management, observability, and security in your Kubernetes environment. The next steps are to enable sidecar injection in your application namespaces and configure traffic management policies to take full advantage of the service mesh capabilities.
