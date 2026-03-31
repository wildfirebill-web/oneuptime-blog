# How to Use Dapr with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Terraform, Infrastructure as Code, Kubernetes, Automation

Description: Provision Dapr installations, Kubernetes clusters, and Dapr component dependencies using Terraform for repeatable, infrastructure-as-code deployments.

---

## Overview

Terraform is the industry standard for infrastructure-as-code. By using Terraform to provision Kubernetes clusters, install Dapr via Helm, and create cloud resources like Redis, PostgreSQL, and messaging services, you achieve fully reproducible Dapr platform deployments.

## Prerequisites

- Terraform v1.5+ installed
- Cloud provider credentials configured
- `kubectl` installed
- Helm Terraform provider

## Terraform Project Structure

```text
dapr-terraform/
  main.tf
  variables.tf
  outputs.tf
  modules/
    dapr/
      main.tf
      variables.tf
    redis/
      main.tf
```

## Installing Dapr with Terraform

Use the Helm Terraform provider to install Dapr:

```hcl
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}

provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "dapr" {
  name             = "dapr"
  repository       = "https://dapr.github.io/helm-charts/"
  chart            = "dapr"
  version          = "1.13.0"
  namespace        = "dapr-system"
  create_namespace = true

  set {
    name  = "global.ha.enabled"
    value = var.ha_enabled
  }

  set {
    name  = "global.logAsJson"
    value = "true"
  }

  wait    = true
  timeout = 300
}
```

## Deploying Dapr Components with Terraform

Use the Kubernetes provider to create Dapr component resources:

```hcl
resource "kubernetes_manifest" "dapr_redis_statestore" {
  manifest = {
    apiVersion = "dapr.io/v1alpha1"
    kind       = "Component"
    metadata = {
      name      = "statestore"
      namespace = "default"
    }
    spec = {
      type    = "state.redis"
      version = "v1"
      metadata = [
        {
          name  = "redisHost"
          value = "${module.redis.host}:6379"
        },
        {
          name = "redisPassword"
          secretKeyRef = {
            name = kubernetes_secret.redis_credentials.metadata[0].name
            key  = "password"
          }
        }
      ]
    }
  }

  depends_on = [helm_release.dapr, module.redis]
}
```

## Provisioning Redis for Dapr

```hcl
module "redis" {
  source = "./modules/redis"

  name               = "dapr-redis"
  resource_group     = azurerm_resource_group.main.name
  location           = var.location
  sku_name           = "Standard"
  capacity           = 1
}

resource "kubernetes_secret" "redis_credentials" {
  metadata {
    name      = "redis-credentials"
    namespace = "default"
  }
  data = {
    password = module.redis.primary_access_key
  }
}
```

## Variables and Outputs

```hcl
variable "ha_enabled" {
  description = "Enable Dapr HA mode"
  type        = bool
  default     = false
}

output "dapr_dashboard_url" {
  description = "URL to access Dapr dashboard"
  value       = "http://localhost:8080 (after port-forward dapr-dashboard)"
}
```

## Running the Deployment

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Verify Dapr installation
kubectl get pods -n dapr-system
```

## Summary

Terraform enables fully automated, version-controlled Dapr platform deployments. By combining the Helm provider for Dapr installation with the Kubernetes provider for component configuration and cloud provider modules for backing services, you achieve reproducible environments that can be consistently deployed across development, staging, and production.
