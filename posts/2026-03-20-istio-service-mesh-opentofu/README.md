# Deploying Istio Service Mesh with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Istio, Kubernetes, OpenTofu, Service Mesh, Observability

Description: Learn how to deploy and configure the Istio service mesh on Kubernetes using OpenTofu, including traffic management, mTLS, and observability setup.

## What is Istio?

Istio is an open-source service mesh that provides traffic management, mutual TLS (mTLS) authentication, observability, and policy enforcement for microservices. It uses sidecar proxies (Envoy) injected alongside application containers to intercept and control all network traffic.

## Why Use OpenTofu for Istio?

OpenTofu allows you to manage Istio installations and configurations as code - versioned, reproducible, and auditable. You can use the Helm provider for installation and the Kubernetes provider for Istio resources.

## Step 1: Install Istio with Helm

```hcl
resource "helm_release" "istio_base" {
  name             = "istio-base"
  repository       = "https://istio-release.storage.googleapis.com/charts"
  chart            = "base"
  namespace        = "istio-system"
  create_namespace = true
  version          = "1.21.0"
}

resource "helm_release" "istiod" {
  name       = "istiod"
  repository = "https://istio-release.storage.googleapis.com/charts"
  chart      = "istiod"
  namespace  = "istio-system"
  version    = "1.21.0"

  depends_on = [helm_release.istio_base]

  set {
    name  = "pilot.traceSampling"
    value = "100"
  }
}
```

## Step 2: Enable Sidecar Injection

```hcl
resource "kubernetes_namespace" "app" {
  metadata {
    name = "production"
    labels = {
      "istio-injection" = "enabled"
    }
  }
}
```

## Step 3: Deploy the Ingress Gateway

```hcl
resource "helm_release" "istio_ingress" {
  name       = "istio-ingress"
  repository = "https://istio-release.storage.googleapis.com/charts"
  chart      = "gateway"
  namespace  = "istio-system"
  version    = "1.21.0"

  depends_on = [helm_release.istiod]
}
```

## Step 4: Configure Traffic Management

```hcl
resource "kubernetes_manifest" "virtual_service" {
  manifest = {
    apiVersion = "networking.istio.io/v1alpha3"
    kind       = "VirtualService"
    metadata = {
      name      = "app-vs"
      namespace = "production"
    }
    spec = {
      hosts    = ["app.example.com"]
      gateways = ["app-gateway"]
      http = [{
        route = [
          {
            destination = {
              host   = "app-v1"
              subset = "v1"
            }
            weight = 90
          },
          {
            destination = {
              host   = "app-v1"
              subset = "v2"
            }
            weight = 10
          }
        ]
      }]
    }
  }
}
```

## Step 5: Enable Mutual TLS

```hcl
resource "kubernetes_manifest" "peer_authentication" {
  manifest = {
    apiVersion = "security.istio.io/v1beta1"
    kind       = "PeerAuthentication"
    metadata = {
      name      = "default"
      namespace = "production"
    }
    spec = {
      mtls = {
        mode = "STRICT"
      }
    }
  }
}
```

## Step 6: Install Observability Tools

```hcl
resource "helm_release" "kiali" {
  name             = "kiali-server"
  repository       = "https://kiali.org/helm-charts"
  chart            = "kiali-server"
  namespace        = "istio-system"
  create_namespace = false
  version          = "1.82.0"

  set {
    name  = "auth.strategy"
    value = "anonymous"
  }
}
```

## Verifying the Installation

```bash
kubectl get pods -n istio-system
istioctl verify-install
istioctl analyze
```

## Best Practices

1. **Enable strict mTLS** across the mesh for zero-trust networking
2. **Use namespaced PeerAuthentication** rather than mesh-wide for gradual rollout
3. **Set resource limits** on Istiod and sidecars to prevent resource starvation
4. **Monitor proxy overhead** - sidecar injection adds latency and CPU usage
5. **Use Kiali** for visual traffic flow debugging

## Conclusion

OpenTofu makes Istio deployments reproducible and configuration-driven. By combining Helm releases for installation with Kubernetes manifest resources for Istio CRDs, you get a fully declarative service mesh setup that integrates naturally with your infrastructure pipeline.
