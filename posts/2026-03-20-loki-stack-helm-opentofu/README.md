# How to Deploy Loki Stack on Kubernetes with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Loki, Grafana, Promtail, Logging, OpenTofu, Helm, Observability

Description: Learn how to deploy the Grafana Loki stack on Kubernetes using OpenTofu and Helm for centralized log aggregation, querying, and visualization with Grafana.

## Overview

Grafana Loki is a log aggregation system inspired by Prometheus. Unlike Elasticsearch, Loki indexes only labels and stores log chunks in object storage, making it cost-effective for large-scale deployments. OpenTofu deploys the full Loki stack - Loki, Promtail, and Grafana - via Helm.

## Step 1: Deploy Loki with Helm

```hcl
# main.tf - Deploy Loki via Grafana Helm chart

resource "helm_release" "loki" {
  name             = "loki"
  repository       = "https://grafana.github.io/helm-charts"
  chart            = "loki"
  version          = "5.47.0"
  namespace        = "monitoring"
  create_namespace = true

  values = [yamlencode({
    loki = {
      # Single binary mode for smaller clusters
      # Use distributed mode (SimpleScalable) for production
      deploymentMode = "SingleBinary"

      auth_enabled = false

      storage = {
        type = "s3"
        s3 = {
          region = "us-east-1"
          bucketnames = "${aws_s3_bucket.loki_chunks.id}"
        }
      }

      # Schema configuration
      schemaConfig = {
        configs = [{
          from   = "2024-01-01"
          store  = "tsdb"
          object_store = "s3"
          schema = "v13"
          index = {
            prefix = "loki_index_"
            period = "24h"
          }
        }]
      }

      limits_config = {
        retention_period            = "744h"  # 31 days
        ingestion_rate_mb           = 10
        ingestion_burst_size_mb     = 20
        max_query_parallelism       = 32
        max_streams_per_user        = 10000
      }
    }

    singleBinary = {
      replicas = 1

      persistence = {
        size         = "50Gi"
        storageClass = "gp3"
      }

      resources = {
        requests = { cpu = "500m", memory = "512Mi" }
        limits   = { cpu = "2000m", memory = "2Gi" }
      }

      # Service account with IRSA for S3 access
      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = aws_iam_role.loki.arn
        }
      }
    }

    # Disable built-in Grafana (use existing)
    grafana = {
      enabled = false
    }
  })]
}
```

## Step 2: S3 Bucket and IAM Role for Loki

```hcl
# S3 bucket for Loki log chunks
resource "aws_s3_bucket" "loki_chunks" {
  bucket = "my-cluster-loki-chunks"
}

resource "aws_s3_bucket_lifecycle_configuration" "loki" {
  bucket = aws_s3_bucket.loki_chunks.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"
    expiration {
      days = 31
    }
  }
}

resource "aws_iam_role" "loki" {
  name = "loki"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.eks.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${local.oidc_provider}:sub" = "system:serviceaccount:monitoring:loki"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy" "loki_s3" {
  role = aws_iam_role.loki.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"]
      Resource = [aws_s3_bucket.loki_chunks.arn, "${aws_s3_bucket.loki_chunks.arn}/*"]
    }]
  })
}
```

## Step 3: Deploy Promtail for Log Collection

```hcl
# Promtail DaemonSet to collect logs from all nodes
resource "helm_release" "promtail" {
  name       = "promtail"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "promtail"
  version    = "6.15.5"
  namespace  = "monitoring"

  depends_on = [helm_release.loki]

  values = [yamlencode({
    config = {
      clients = [{
        url = "http://loki:3100/loki/api/v1/push"
      }]

      # Pipeline stages for log parsing
      snippets = {
        extraScrapeConfigs = <<-SCRAPE
          - job_name: kubernetes-pods
            kubernetes_sd_configs:
              - role: pod
            pipeline_stages:
              - cri: {}
              - labeldrop:
                  - filename
                  - stream
              - labels:
                  app:
                  namespace:
                  pod:
        SCRAPE
      }
    }

    resources = {
      requests = { cpu = "50m", memory = "64Mi" }
      limits   = { cpu = "200m", memory = "128Mi" }
    }
  })]
}
```

## Step 4: Configure Grafana Data Source

```hcl
# Add Loki as a data source in Grafana
resource "kubernetes_config_map" "loki_datasource" {
  metadata {
    name      = "loki-datasource"
    namespace = "monitoring"
    labels = {
      "grafana_datasource" = "1"
    }
  }

  data = {
    "loki-datasource.yaml" = yamlencode({
      apiVersion = 1
      datasources = [{
        name      = "Loki"
        type      = "loki"
        url       = "http://loki:3100"
        access    = "proxy"
        isDefault = false
        jsonData = {
          maxLines = 1000
          derivedFields = [{
            matcherRegex    = "traceID=(\\w+)"
            name            = "TraceID"
            url             = "$${__value.raw}"
            datasourceUid   = "tempo"
          }]
        }
      }]
    })
  }
}
```

## Summary

Grafana Loki deployed with OpenTofu provides cost-efficient log aggregation using S3 as the storage backend. Promtail runs as a DaemonSet, collecting logs from all pods and enriching them with Kubernetes labels. The label-based indexing model means log storage costs scale with the number of unique label combinations rather than log volume, making Loki significantly cheaper than full-text search solutions for large fleets.
