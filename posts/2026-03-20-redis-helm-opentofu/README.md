# How to Deploy Redis on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Redis, Caching, OpenTofu, Helm, High Availability

Description: Learn how to deploy Redis on Kubernetes using OpenTofu and the Bitnami Helm chart with Sentinel-based high availability, persistent storage, and TLS encryption.

## Overview

Redis is an in-memory data store used for caching, session management, and pub/sub messaging. OpenTofu deploys Redis via the Bitnami Helm chart with Sentinel HA, optional persistence, and password authentication.

## Step 1: Deploy Redis with Helm

```hcl
# main.tf - Deploy Redis via Bitnami Helm chart
resource "random_password" "redis" {
  length  = 32
  special = false
}

resource "kubernetes_secret" "redis_auth" {
  metadata {
    name      = "redis-auth"
    namespace = "redis"
  }

  data = {
    redis-password = random_password.redis.result
  }
}

resource "helm_release" "redis" {
  name             = "redis"
  repository       = "https://charts.bitnami.com/bitnami"
  chart            = "redis"
  version          = "18.16.0"
  namespace        = "redis"
  create_namespace = true

  values = [yamlencode({
    architecture = "replication"  # standalone or replication

    auth = {
      enabled             = true
      existingSecret      = kubernetes_secret.redis_auth.metadata[0].name
      existingSecretPasswordKey = "redis-password"
    }

    master = {
      replicaCount = 1

      persistence = {
        enabled      = true
        size         = "10Gi"
        storageClass = "gp3"
      }

      resources = {
        requests = { memory = "256Mi", cpu = "250m" }
        limits   = { memory = "512Mi", cpu = "500m" }
      }

      # Redis configuration overrides
      configuration = <<-REDIS_CONF
        maxmemory 400mb
        maxmemory-policy allkeys-lru
        save 900 1
        save 300 10
      REDIS_CONF
    }

    replica = {
      replicaCount = 2

      persistence = {
        enabled = true
        size    = "10Gi"
      }

      resources = {
        requests = { memory = "256Mi", cpu = "250m" }
        limits   = { memory = "512Mi", cpu = "500m" }
      }
    }

    # Sentinel for HA failover
    sentinel = {
      enabled          = true
      masterSet        = "mymaster"
      quorum           = 2
      downAfterMilliseconds = 5000
      failoverTimeout  = 10000
    }

    metrics = {
      enabled = true
      serviceMonitor = {
        enabled = true
      }
    }
  })]
}
```

## Step 2: Redis Cluster Mode (for large datasets)

```hcl
# Redis Cluster for sharded high-throughput workloads
resource "helm_release" "redis_cluster" {
  name             = "redis-cluster"
  repository       = "https://charts.bitnami.com/bitnami"
  chart            = "redis-cluster"
  version          = "10.2.0"
  namespace        = "redis"
  create_namespace = true

  values = [yamlencode({
    cluster = {
      nodes   = 6  # 3 masters + 3 replicas
      replicas = 1
    }

    auth = {
      enabled        = true
      existingSecret = kubernetes_secret.redis_auth.metadata[0].name
    }

    persistence = {
      enabled = true
      size    = "10Gi"
    }

    redis = {
      resources = {
        requests = { memory = "256Mi", cpu = "250m" }
        limits   = { memory = "1Gi", cpu = "500m" }
      }
    }
  })]
}
```

## Step 3: PodDisruptionBudget for Redis

```hcl
# Ensure quorum is maintained during disruptions
resource "kubernetes_pod_disruption_budget_v1" "redis" {
  metadata {
    name      = "redis-pdb"
    namespace = "redis"
  }

  spec {
    max_unavailable = "1"
    selector {
      match_labels = {
        "app.kubernetes.io/name"      = "redis"
        "app.kubernetes.io/component" = "replica"
      }
    }
  }
}
```

## Step 4: Outputs

```hcl
output "redis_host" {
  value = "redis-master.redis.svc.cluster.local"
}

output "redis_sentinel_host" {
  value = "redis.redis.svc.cluster.local"
}

output "redis_port" {
  value = 6379
}
```

## Summary

Redis deployed with OpenTofu using Bitnami's Helm chart provides production-grade caching with Sentinel-based automatic failover. The Sentinel architecture monitors masters and promotes replicas automatically when a master fails. For larger datasets requiring horizontal sharding, Redis Cluster mode distributes data across multiple master nodes while maintaining high availability.
