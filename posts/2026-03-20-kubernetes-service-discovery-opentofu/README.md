# How to Configure Kubernetes Service Discovery with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Kubernetes, Service Discovery, DNS, Microservice, Infrastructure as Code

Description: Learn how to configure Kubernetes Services, Endpoints, and ExternalName resources for service discovery using OpenTofu's Kubernetes provider.

## Introduction

Kubernetes has built-in service discovery through Services and DNS. Every Service gets a DNS name following the pattern `service.namespace.svc.cluster.local`. OpenTofu manages Services, headless services, and ExternalName services as code for consistent multi-environment configurations.

## Creating a ClusterIP Service

```hcl
resource "kubernetes_service_v1" "payments" {
  metadata {
    name      = "payments"
    namespace = kubernetes_namespace_v1.app.metadata[0].name
    labels = {
      app = "payments"
    }
  }

  spec {
    selector = {
      app = "payments"
    }

    port {
      name        = "http"
      port        = 80
      target_port = 8080
      protocol    = "TCP"
    }

    type = "ClusterIP"
  }
}
```

## Headless Service for StatefulSets

Headless services allow direct DNS resolution to individual pod IPs.

```hcl
resource "kubernetes_service_v1" "postgres_headless" {
  metadata {
    name      = "postgres"
    namespace = kubernetes_namespace_v1.app.metadata[0].name
  }

  spec {
    cluster_ip = "None"  # headless service – no virtual IP

    selector = {
      app = "postgres"
    }

    port {
      name     = "postgres"
      port     = 5432
      protocol = "TCP"
    }
  }
}
```

## ExternalName Service

Route internal traffic to an external hostname (e.g., an RDS endpoint).

```hcl
resource "kubernetes_service_v1" "database" {
  metadata {
    name      = "database"
    namespace = kubernetes_namespace_v1.app.metadata[0].name
  }

  spec {
    type          = "ExternalName"
    external_name = var.rds_endpoint  # e.g., mydb.xxxxxx.us-east-1.rds.amazonaws.com
  }
}
```

## Custom Endpoints

Register external services with custom IP addresses.

```hcl
resource "kubernetes_service_v1" "legacy_api" {
  metadata {
    name      = "legacy-api"
    namespace = kubernetes_namespace_v1.app.metadata[0].name
  }
  spec {
    port {
      port     = 80
      protocol = "TCP"
    }
  }
}

resource "kubernetes_endpoints_v1" "legacy_api" {
  metadata {
    name      = kubernetes_service_v1.legacy_api.metadata[0].name
    namespace = kubernetes_namespace_v1.app.metadata[0].name
  }

  subset {
    address {
      ip = "192.168.1.100"
    }
    address {
      ip = "192.168.1.101"
    }

    port {
      port     = 8080
      protocol = "TCP"
    }
  }
}
```

## Namespace

```hcl
resource "kubernetes_namespace_v1" "app" {
  metadata {
    name = var.namespace
  }
}
```

## Service DNS Names

Once created, services are accessible at these DNS names within the cluster:

- Same namespace: `payments` or `payments.default`
- Cross namespace: `payments.production.svc.cluster.local`
- External via ExternalName: resolves to the RDS endpoint CNAME

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Kubernetes service discovery relies on Services and DNS, both of which can be fully managed with OpenTofu. By using ClusterIP services for internal communication, headless services for StatefulSets, and ExternalName services for external resources, you create a consistent service discovery layer across all environments.
