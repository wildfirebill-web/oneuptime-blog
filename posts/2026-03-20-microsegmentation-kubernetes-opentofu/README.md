# How to Set Up Microsegmentation with OpenTofu on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Microsegmentation, Network Policy, OpenTofu, Cilium, Security

Description: Learn how to implement Kubernetes microsegmentation using OpenTofu with Network Policies to enforce pod-level traffic controls for zero trust within the cluster.

## Overview

Kubernetes microsegmentation uses Network Policies to control traffic between pods at Layer 3/4. By default, all pods can communicate freely. OpenTofu provisions Network Policies with a default-deny stance and explicit allow rules per service.

## Step 1: Default Deny All (Namespace-Level)

```hcl
# main.tf - Default deny all ingress and egress per namespace
resource "kubernetes_network_policy" "default_deny" {
  for_each = toset(["production", "staging", "database"])

  metadata {
    name      = "default-deny-all"
    namespace = each.value
  }

  spec {
    pod_selector {}  # Applies to all pods

    # Empty policy types mean deny all
    policy_types = ["Ingress", "Egress"]
  }
}

# Allow DNS egress (required for all pods)
resource "kubernetes_network_policy" "allow_dns" {
  for_each = toset(["production", "staging", "database"])

  metadata {
    name      = "allow-dns-egress"
    namespace = each.value
  }

  spec {
    pod_selector {}

    policy_types = ["Egress"]

    egress {
      ports {
        port     = "53"
        protocol = "UDP"
      }
      ports {
        port     = "53"
        protocol = "TCP"
      }

      to {
        namespace_selector {
          match_labels = {
            "kubernetes.io/metadata.name" = "kube-system"
          }
        }
        pod_selector {
          match_labels = { "k8s-app" = "kube-dns" }
        }
      }
    }
  }
}
```

## Step 2: Service-Level Allow Rules

```hcl
# Allow frontend to call API
resource "kubernetes_network_policy" "api_allow_frontend" {
  metadata {
    name      = "api-allow-frontend"
    namespace = "production"
  }

  spec {
    pod_selector {
      match_labels = { app = "api" }
    }

    policy_types = ["Ingress"]

    ingress {
      from {
        namespace_selector {
          match_labels = {
            "kubernetes.io/metadata.name" = "production"
          }
        }
        pod_selector {
          match_labels = { app = "frontend" }
        }
      }

      ports {
        port     = "8080"
        protocol = "TCP"
      }
    }
  }
}

# Allow API to access database
resource "kubernetes_network_policy" "db_allow_api" {
  metadata {
    name      = "db-allow-api"
    namespace = "database"
  }

  spec {
    pod_selector {
      match_labels = { app = "postgresql" }
    }

    policy_types = ["Ingress"]

    ingress {
      from {
        namespace_selector {
          match_labels = {
            "kubernetes.io/metadata.name" = "production"
          }
        }
        pod_selector {
          match_labels = { app = "api" }
        }
      }

      ports {
        port     = "5432"
        protocol = "TCP"
      }
    }
  }
}
```

## Step 3: Monitoring Access (Cross-Namespace)

```hcl
# Allow Prometheus to scrape metrics from all namespaces
resource "kubernetes_network_policy" "allow_prometheus_scrape" {
  for_each = toset(["production", "staging"])

  metadata {
    name      = "allow-prometheus-scrape"
    namespace = each.value
  }

  spec {
    pod_selector {}  # All pods in namespace

    policy_types = ["Ingress"]

    ingress {
      from {
        namespace_selector {
          match_labels = {
            "kubernetes.io/metadata.name" = "monitoring"
          }
        }
        pod_selector {
          match_labels = {
            "app.kubernetes.io/name" = "prometheus"
          }
        }
      }

      ports {
        port     = "9090"
        protocol = "TCP"
      }
    }
  }
}
```

## Step 4: Cilium Network Policy (Extended Layer 7)

```hcl
# Cilium L7 policy for HTTP method-level control
resource "kubernetes_manifest" "cilium_l7_policy" {
  manifest = {
    apiVersion = "cilium.io/v2"
    kind       = "CiliumNetworkPolicy"
    metadata = {
      name      = "api-l7-policy"
      namespace = "production"
    }
    spec = {
      endpointSelector = {
        matchLabels = { app = "api" }
      }
      ingress = [{
        fromEndpoints = [{
          matchLabels = { app = "frontend" }
        }]
        toPorts = [{
          ports = [{ port = "8080", protocol = "TCP" }]
          rules = {
            http = [
              { method = "GET", path = "/api/.*" },
              { method = "POST", path = "/api/orders" }
            ]
          }
        }]
      }]
    }
  }
}
```

## Summary

Kubernetes microsegmentation with OpenTofu starts from a default-deny stance and adds explicit allow policies per service. This ensures newly deployed pods are isolated until a Network Policy explicitly allows their traffic. Cilium extends Layer 3/4 Network Policies to Layer 7, enabling HTTP method and path-based controls for API access within the cluster, providing the granularity needed for true zero trust within Kubernetes.
