# How to Create Kubernetes Vertical Pod Autoscalers with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, VPA, Vertical Pod Autoscaler, OpenTofu, Resource Optimization, Infrastructure

Description: Learn how to create Kubernetes Vertical Pod Autoscalers (VPA) with OpenTofu to automatically right-size container CPU and memory requests for optimal resource utilization.

## Overview

Kubernetes Vertical Pod Autoscaler automatically adjusts CPU and memory requests for containers based on historical usage patterns. VPA eliminates the need to manually tune resource requests and helps prevent OOMKilled pods. OpenTofu manages VPA objects with different update modes.

## Step 1: Install VPA CRDs

```hcl
# main.tf - Deploy VPA using Helm (requires VPA admission webhook)
resource "helm_release" "vpa" {
  name       = "vpa"
  repository = "https://charts.fairwinds.com/stable"
  chart      = "vpa"
  namespace  = "kube-system"
  version    = "4.4.6"

  set {
    name  = "admissionController.enabled"
    value = "true"
  }
}
```

## Step 2: VPA in Recommendation Mode (Off)

```hcl
# VPA in "Off" mode - provides recommendations without applying them
resource "kubernetes_manifest" "web_app_vpa_off" {
  manifest = {
    apiVersion = "autoscaling.k8s.io/v1"
    kind       = "VerticalPodAutoscaler"
    metadata = {
      name      = "web-app-vpa-recommendations"
      namespace = "production"
    }
    spec = {
      targetRef = {
        apiVersion = "apps/v1"
        kind       = "Deployment"
        name       = "web-app"
      }
      updatePolicy = {
        # "Off" - only generates recommendations, doesn't change pods
        updateMode = "Off"
      }
      resourcePolicy = {
        containerPolicies = [
          {
            containerName = "web-app"
            minAllowed = {
              cpu    = "100m"
              memory = "128Mi"
            }
            maxAllowed = {
              cpu    = "4"
              memory = "8Gi"
            }
          }
        ]
      }
    }
  }

  depends_on = [helm_release.vpa]
}
```

## Step 3: VPA in Auto Mode (Applies Recommendations)

```hcl
# VPA in "Auto" mode - updates pods with new resource requests
resource "kubernetes_manifest" "api_vpa_auto" {
  manifest = {
    apiVersion = "autoscaling.k8s.io/v1"
    kind       = "VerticalPodAutoscaler"
    metadata = {
      name      = "api-service-vpa"
      namespace = "production"
    }
    spec = {
      targetRef = {
        apiVersion = "apps/v1"
        kind       = "Deployment"
        name       = "api-service"
      }
      updatePolicy = {
        # "Auto" - evicts pods and restarts with updated resource requests
        updateMode = "Auto"
      }
      resourcePolicy = {
        containerPolicies = [
          {
            containerName          = "api"
            controlledResources    = ["cpu", "memory"]
            # Define bounds for VPA recommendations
            minAllowed = {
              cpu    = "50m"
              memory = "64Mi"
            }
            maxAllowed = {
              cpu    = "2"
              memory = "4Gi"
            }
          }
        ]
      }
    }
  }
}
```

## Step 4: VPA for InitContainers

```hcl
# VPA managing both init containers and main containers
resource "kubernetes_manifest" "full_vpa" {
  manifest = {
    apiVersion = "autoscaling.k8s.io/v1"
    kind       = "VerticalPodAutoscaler"
    metadata = {
      name      = "full-app-vpa"
      namespace = "production"
    }
    spec = {
      targetRef = {
        apiVersion = "apps/v1"
        kind       = "Deployment"
        name       = "full-app"
      }
      updatePolicy = {
        updateMode = "Initial"  # Only apply recommendations at pod creation
      }
      resourcePolicy = {
        containerPolicies = [
          {
            containerName = "*"  # Apply to all containers
            minAllowed = { cpu = "50m", memory = "64Mi" }
            maxAllowed  = { cpu = "4", memory = "8Gi" }
          }
        ]
      }
    }
  }
}
```

## Summary

Kubernetes VPA with OpenTofu automates resource request optimization. Start in "Off" mode to collect recommendations without disrupting workloads, then gradually move to "Initial" (apply at pod creation only) and "Auto" (actively right-size running pods). VPA and HPA can work together—use VPA for right-sizing and HPA for scaling.
