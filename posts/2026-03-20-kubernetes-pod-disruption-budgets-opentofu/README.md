# How to Create Kubernetes Pod Disruption Budgets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, PDB, Pod Disruption Budget, OpenTofu, High Availability, Infrastructure

Description: Learn how to create Kubernetes Pod Disruption Budgets (PDB) with OpenTofu to ensure minimum availability during voluntary disruptions like node drains and cluster upgrades.

## Overview

Pod Disruption Budgets (PDBs) limit the number of pods that can be voluntarily disrupted at a given time. They protect application availability during node maintenance, cluster upgrades, and scale-down operations. OpenTofu manages PDBs alongside deployments.

## Step 1: Create PDB with minAvailable

```hcl
# main.tf - Ensure at least 3 pods are always available
resource "kubernetes_pod_disruption_budget_v1" "web_app_pdb" {
  metadata {
    name      = "web-app-pdb"
    namespace = "production"
  }

  spec {
    # Minimum number of pods that must be available
    min_available = 3

    selector {
      match_labels = {
        app = "web-app"
      }
    }
  }
}
```

## Step 2: PDB with Percentage-Based Availability

```hcl
# Keep at least 80% of pods available during disruptions
resource "kubernetes_pod_disruption_budget_v1" "api_pdb" {
  metadata {
    name      = "api-service-pdb"
    namespace = "production"
  }

  spec {
    min_available = "80%"  # At least 80% must be available

    selector {
      match_labels = {
        app     = "api-service"
        version = "stable"
      }
    }
  }
}
```

## Step 3: PDB with maxUnavailable

```hcl
# Allow at most 1 pod to be unavailable at a time
resource "kubernetes_pod_disruption_budget_v1" "database_pdb" {
  metadata {
    name      = "postgres-pdb"
    namespace = "production"
  }

  spec {
    # Maximum number of pods that can be disrupted simultaneously
    max_unavailable = 1

    selector {
      match_labels = {
        app = "postgres"
      }
    }
  }
}
```

## Step 4: PDB for StatefulSet

```hcl
# PDB for a 3-replica StatefulSet - keep quorum (minimum 2)
resource "kubernetes_pod_disruption_budget_v1" "redis_pdb" {
  metadata {
    name      = "redis-pdb"
    namespace = "production"
  }

  spec {
    min_available = 2  # Maintains quorum for 3-node cluster

    selector {
      match_labels = {
        app = "redis"
      }
    }
  }
}
```

## Step 5: Combined with Deployment

```hcl
# Deployment with matching PDB for end-to-end availability guarantee
resource "kubernetes_deployment_v1" "web_app" {
  metadata {
    name      = "web-app"
    namespace = "production"
  }

  spec {
    replicas = 5

    selector {
      match_labels = { app = "web-app" }
    }

    template {
      metadata {
        labels = { app = "web-app" }
      }

      spec {
        container {
          name  = "web-app"
          image = "myregistry/web-app:latest"

          port {
            container_port = 8080
          }
        }
      }
    }
  }
}

# Matching PDB for the deployment
resource "kubernetes_pod_disruption_budget_v1" "web_app_pdb_v2" {
  metadata {
    name      = "web-app-pdb"
    namespace = "production"
  }

  spec {
    max_unavailable = "20%"  # Allow 1 of 5 pods to be disrupted

    selector {
      match_labels = {
        app = "web-app"
      }
    }
  }
}
```

## Summary

Kubernetes Pod Disruption Budgets with OpenTofu guarantee minimum application availability during voluntary cluster operations. Use `min_available` for absolute counts, percentages for proportional availability, and `max_unavailable` for more permissive budgets. PDBs are essential for high-availability applications in clusters that undergo frequent node maintenance or upgrades.
