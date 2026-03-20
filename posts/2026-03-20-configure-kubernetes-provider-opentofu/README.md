# How to Configure the Kubernetes Provider in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Kubernetes, Provider Configuration, Infrastructure as Code, K8s

Description: Learn how to configure the Kubernetes provider in OpenTofu to manage Kubernetes resources declaratively alongside your cloud infrastructure.

## Introduction

The Kubernetes provider allows OpenTofu to manage Kubernetes resources-Deployments, Services, ConfigMaps, and more-using the same HCL syntax as cloud resources. You can configure it from a kubeconfig file, inline credentials, or dynamically from an EKS/AKS/GKE cluster resource.

## Configuration from Kubeconfig File

```hcl
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "my-cluster-context"
}
```

## Dynamic Configuration from EKS

```hcl
data "aws_eks_cluster" "main" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "main" {
  name = var.cluster_name
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}
```

## Dynamic Configuration from AKS

```hcl
provider "kubernetes" {
  host = azurerm_kubernetes_cluster.main.kube_config[0].host

  client_certificate     = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.main.kube_config[0].cluster_ca_certificate)
}
```

## Dynamic Configuration from GKE

```hcl
data "google_client_config" "default" {}

provider "kubernetes" {
  host  = "https://${google_container_cluster.main.endpoint}"
  token = data.google_client_config.default.access_token

  cluster_ca_certificate = base64decode(
    google_container_cluster.main.master_auth[0].cluster_ca_certificate
  )
}
```

## Multiple Clusters

```hcl
provider "kubernetes" {
  alias          = "dev"
  config_path    = "~/.kube/config"
  config_context = "dev-cluster"
}

provider "kubernetes" {
  alias          = "prod"
  config_path    = "~/.kube/config"
  config_context = "prod-cluster"
}

resource "kubernetes_namespace" "monitoring" {
  provider = kubernetes.prod

  metadata {
    name = "monitoring"
  }
}
```

## Conclusion

The Kubernetes provider is most powerful when configured dynamically from a just-created cloud cluster resource-this ensures the provider always uses fresh, valid credentials. Avoid hardcoding kubeconfig paths in shared configurations; prefer dynamic token-based authentication for CI/CD pipelines.
