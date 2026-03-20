# How to Deploy Istio Service Mesh on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Istio, Service Mesh, OpenTofu, Helm, mTLS, Traffic Management

Description: Learn how to deploy Istio service mesh on Kubernetes using OpenTofu and Helm with automatic mTLS, traffic management, observability, and canary deployments.

## Overview

Istio provides a full-featured service mesh with automatic mTLS between services, fine-grained traffic control, distributed tracing, and access policies. OpenTofu deploys Istio using the official Helm charts in the recommended multi-chart approach.

## Step 1: Deploy Istio Base and Istiod

```hcl
# main.tf - Deploy Istio via Helm (recommended multi-chart approach)
resource "helm_release" "istio_base" {
  name             = "istio-base"
  repository       = "https://istio-release.storage.googleapis.com/charts"
  chart            = "base"
  version          = "1.20.0"
  namespace        = "istio-system"
  create_namespace = true
}

resource "helm_release" "istiod" {
  name       = "istiod"
  repository = "https://istio-release.storage.googleapis.com/charts"
  chart      = "istiod"
  version    = "1.20.0"
  namespace  = "istio-system"

  depends_on = [helm_release.istio_base]

  values = [yamlencode({
    pilot = {
      resources = {
        requests = { cpu = "200m", memory = "128Mi" }
        limits   = { cpu = "500m", memory = "512Mi" }
      }

      autoscaleMin = 2
      autoscaleMax = 5
    }

    meshConfig = {
      # Enable access logging
      accessLogFile = "/dev/stdout"

      # Enable distributed tracing
      enableTracing = true
      defaultConfig = {
        tracing = {
          sampling = 10  # 10% sampling rate
          zipkin = {
            address = "zipkin.istio-system:9411"
          }
        }
      }

      # Default mTLS mode
      mtls = {
        mode = "STRICT"
      }
    }
  })]
}

# Istio Ingress Gateway
resource "helm_release" "istio_ingress" {
  name       = "istio-ingressgateway"
  repository = "https://istio-release.storage.googleapis.com/charts"
  chart      = "gateway"
  version    = "1.20.0"
  namespace  = "istio-system"

  depends_on = [helm_release.istiod]

  values = [yamlencode({
    service = {
      type = "LoadBalancer"
    }

    autoscaling = {
      minReplicas = 2
      maxReplicas = 10
    }

    resources = {
      requests = { cpu = "100m", memory = "128Mi" }
      limits   = { cpu = "2000m", memory = "1024Mi" }
    }
  })]
}
```

## Step 2: Enable Istio Sidecar Injection

```hcl
# Label namespace to enable automatic sidecar injection
resource "kubernetes_namespace" "production" {
  metadata {
    name = "production"
    labels = {
      "istio-injection" = "enabled"
    }
  }
}
```

## Step 3: Configure Gateway and VirtualService

```hcl
# Istio Gateway resource for ingress traffic
resource "kubernetes_manifest" "gateway" {
  manifest = {
    apiVersion = "networking.istio.io/v1beta1"
    kind       = "Gateway"
    metadata = {
      name      = "app-gateway"
      namespace = "istio-system"
    }
    spec = {
      selector = {
        istio = "ingressgateway"
      }
      servers = [{
        port = {
          number   = 443
          name     = "https"
          protocol = "HTTPS"
        }
        tls = {
          mode              = "SIMPLE"
          credentialName    = "app-tls"
        }
        hosts = ["app.example.com"]
      }]
    }
  }
}

# VirtualService for traffic routing
resource "kubernetes_manifest" "virtual_service" {
  manifest = {
    apiVersion = "networking.istio.io/v1beta1"
    kind       = "VirtualService"
    metadata = {
      name      = "app"
      namespace = "production"
    }
    spec = {
      hosts    = ["app.example.com"]
      gateways = ["istio-system/app-gateway"]
      http = [{
        match = [{ uri = { prefix = "/" } }]
        route = [
          {
            destination = { host = "app-v1", port = { number = 80 } }
            weight = 90
          },
          {
            destination = { host = "app-v2", port = { number = 80 } }
            weight = 10  # 10% canary traffic
          }
        ]
      }]
    }
  }
}
```

## Step 4: PeerAuthentication for mTLS

```hcl
# Enforce strict mTLS across the namespace
resource "kubernetes_manifest" "peer_auth" {
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

# AuthorizationPolicy to restrict access
resource "kubernetes_manifest" "authz_policy" {
  manifest = {
    apiVersion = "security.istio.io/v1beta1"
    kind       = "AuthorizationPolicy"
    metadata = {
      name      = "api-access"
      namespace = "production"
    }
    spec = {
      selector = {
        matchLabels = { app = "api" }
      }
      action = "ALLOW"
      rules = [{
        from = [{
          source = {
            principals = ["cluster.local/ns/production/sa/frontend"]
          }
        }]
        to = [{
          operation = {
            methods = ["GET", "POST"]
            paths   = ["/api/*"]
          }
        }]
      }]
    }
  }
}
```

## Summary

Istio deployed with OpenTofu provides comprehensive service mesh capabilities without application code changes. Automatic mTLS encrypts all service-to-service communication, VirtualService resources enable canary deployments with traffic splitting, and AuthorizationPolicies enforce zero-trust access control based on cryptographic service identity.
