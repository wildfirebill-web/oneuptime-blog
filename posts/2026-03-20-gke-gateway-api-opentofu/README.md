# How to Set Up GKE Gateway API with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GKE, Gateway API, Kubernetes, OpenTofu, Networking, Ingress

Description: Learn how to configure GKE Gateway API with OpenTofu for advanced HTTP routing, TLS termination, and traffic splitting using the Kubernetes Gateway API spec.

## Overview

GKE Gateway API is the next-generation Kubernetes ingress solution, offering more expressive routing rules, multi-tenancy support, and standardized traffic management. OpenTofu enables the Gateway API and deploys Gateway and HTTPRoute resources.

## Step 1: Enable Gateway API on GKE Cluster

```hcl
# main.tf - GKE cluster with Gateway API enabled

resource "google_container_cluster" "gateway_cluster" {
  name     = "gateway-api-cluster"
  location = "us-central1"

  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {}

  # Enable Gateway API
  gateway_api_config {
    channel = "CHANNEL_STANDARD"
  }
}
```

## Step 2: Deploy a Gateway

```hcl
# Deploy a Gateway resource (external HTTPS)
resource "kubernetes_manifest" "external_gateway" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "Gateway"
    metadata = {
      name      = "external-gateway"
      namespace = "default"
    }
    spec = {
      # GKE-managed external Application Load Balancer
      gatewayClassName = "gke-l7-global-external-managed"
      listeners = [
        {
          name     = "https"
          port     = 443
          protocol = "HTTPS"
          tls = {
            mode = "Terminate"
            certificateRefs = [
              {
                kind  = "Secret"
                name  = "tls-secret"
                group = ""
              }
            ]
          }
        }
      ]
    }
  }
}
```

## Step 3: Deploy HTTPRoute Resources

```hcl
# HTTPRoute for the main web application
resource "kubernetes_manifest" "web_app_route" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "HTTPRoute"
    metadata = {
      name      = "web-app-route"
      namespace = "default"
    }
    spec = {
      parentRefs = [
        {
          name = "external-gateway"
        }
      ]
      hostnames = ["app.example.com"]
      rules = [
        {
          matches = [
            {
              path = {
                type  = "PathPrefix"
                value = "/api"
              }
            }
          ]
          backendRefs = [
            {
              name = "api-service"
              port = 8080
            }
          ]
        },
        {
          matches = [
            {
              path = {
                type  = "PathPrefix"
                value = "/"
              }
            }
          ]
          backendRefs = [
            {
              name = "web-service"
              port = 80
            }
          ]
        }
      ]
    }
  }

  depends_on = [kubernetes_manifest.external_gateway]
}
```

## Step 4: Traffic Splitting for Canary Deployments

```hcl
# Canary deployment with traffic splitting
resource "kubernetes_manifest" "canary_route" {
  manifest = {
    apiVersion = "gateway.networking.k8s.io/v1"
    kind       = "HTTPRoute"
    metadata = {
      name      = "canary-route"
      namespace = "default"
    }
    spec = {
      parentRefs = [{ name = "external-gateway" }]
      rules = [
        {
          backendRefs = [
            {
              name   = "app-v1"
              port   = 80
              weight = 90  # 90% to stable version
            },
            {
              name   = "app-v2"
              port   = 80
              weight = 10  # 10% to canary version
            }
          ]
        }
      ]
    }
  }
}
```

## Summary

GKE Gateway API with OpenTofu provides a Kubernetes-native approach to advanced traffic management. HTTPRoutes support path-based routing, header matching, and traffic splitting for canary deployments. The GKE-managed gateway eliminates the need to manage ingress controllers while providing integration with Google Cloud Load Balancing.
