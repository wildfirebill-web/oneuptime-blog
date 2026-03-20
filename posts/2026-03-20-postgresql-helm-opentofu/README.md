# How to Deploy PostgreSQL on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, PostgreSQL, Database, OpenTofu, Helm, StatefulSet, High Availability

Description: Learn how to deploy PostgreSQL on Kubernetes using OpenTofu and the Bitnami Helm chart with replication, persistent storage, connection pooling with PgBouncer, and automated backups.

## Overview

PostgreSQL on Kubernetes with the Bitnami Helm chart provides a production-ready setup with primary-replica replication, persistent storage, and optional PgBouncer connection pooling. OpenTofu manages the full deployment declaratively.

## Step 1: Deploy PostgreSQL with Helm

```hcl
# main.tf - Deploy PostgreSQL via Bitnami Helm chart
resource "random_password" "postgres_admin" {
  length  = 32
  special = false
}

resource "random_password" "postgres_app" {
  length  = 32
  special = false
}

resource "kubernetes_secret" "postgres_auth" {
  metadata {
    name      = "postgres-auth"
    namespace = "postgres"
  }

  data = {
    postgres-password = random_password.postgres_admin.result
    password          = random_password.postgres_app.result
    replication-password = random_password.postgres_admin.result
  }
}

resource "helm_release" "postgresql" {
  name             = "postgresql"
  repository       = "https://charts.bitnami.com/bitnami"
  chart            = "postgresql"
  version          = "14.2.0"
  namespace        = "postgres"
  create_namespace = true

  values = [yamlencode({
    auth = {
      username          = "appuser"
      database          = "appdb"
      existingSecret    = kubernetes_secret.postgres_auth.metadata[0].name
    }

    primary = {
      replicaCount = 1

      persistence = {
        enabled      = true
        size         = "20Gi"
        storageClass = "gp3"
      }

      resources = {
        requests = { memory = "256Mi", cpu = "250m" }
        limits   = { memory = "1Gi", cpu = "1000m" }
      }

      # PostgreSQL configuration
      extendedConfiguration = <<-PG_CONF
        max_connections = 200
        shared_buffers = 256MB
        effective_cache_size = 768MB
        work_mem = 4MB
        maintenance_work_mem = 64MB
        wal_level = replica
        max_wal_senders = 5
      PG_CONF

      initdb = {
        scripts = {
          "init.sql" = <<-SQL
            CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
            CREATE EXTENSION IF NOT EXISTS pgcrypto;
          SQL
        }
      }
    }

    readReplicas = {
      replicaCount = 2

      persistence = {
        enabled = true
        size    = "20Gi"
      }

      resources = {
        requests = { memory = "256Mi", cpu = "250m" }
        limits   = { memory = "1Gi", cpu = "500m" }
      }
    }

    # Enable connection pooling with PgBouncer
    pgbouncer = {
      enabled = true

      poolMode = "transaction"  # session, transaction, or statement

      maxClientConnections  = 400
      defaultPoolSize       = 20
      minPoolSize           = 5
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

## Step 2: PodDisruptionBudget

```hcl
# Protect PostgreSQL replicas during cluster maintenance
resource "kubernetes_pod_disruption_budget_v1" "postgres_replicas" {
  metadata {
    name      = "postgres-replicas-pdb"
    namespace = "postgres"
  }

  spec {
    min_available = 1
    selector {
      match_labels = {
        "app.kubernetes.io/name"      = "postgresql"
        "app.kubernetes.io/component" = "read"
      }
    }
  }
}
```

## Step 3: Outputs

```hcl
output "postgres_primary_host" {
  value       = "postgresql.postgres.svc.cluster.local"
  description = "PostgreSQL primary host for writes"
}

output "postgres_replica_host" {
  value       = "postgresql-read.postgres.svc.cluster.local"
  description = "PostgreSQL read replica host for reads"
}

output "pgbouncer_host" {
  value       = "postgresql-pgbouncer.postgres.svc.cluster.local"
  description = "PgBouncer connection pooler host"
}

output "postgres_port" {
  value = 5432
}
```

## Summary

PostgreSQL deployed with OpenTofu on Kubernetes using Bitnami's Helm chart provides a reliable relational database with streaming replication and optional PgBouncer connection pooling. The read replica endpoint allows applications to distribute read load while writing to the primary. PgBouncer in transaction pooling mode dramatically reduces connection overhead for high-concurrency workloads.
