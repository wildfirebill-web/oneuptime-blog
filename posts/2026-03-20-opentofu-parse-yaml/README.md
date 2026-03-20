# How to Parse YAML Files for Configuration in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, YAML, Yamldecode, Configuration, Helm, Kubernetes

Description: Learn how to parse YAML configuration files in OpenTofu using yamldecode to drive Kubernetes and Helm deployments from YAML-based configuration.

## Overview

OpenTofu's `yamldecode` function parses YAML into HCL data structures, enabling infrastructure configuration in YAML format. This is particularly useful when working with Kubernetes and Helm where YAML is the native format.

## Step 1: Basic YAML Parsing

```hcl
# main.tf - Parse YAML configuration files

locals {
  # Parse a YAML configuration file
  app_config = yamldecode(file("${path.module}/config/app.yaml"))
}

# config/app.yaml:
# environments:
#   production:
#     replicas: 5
#     resources:
#       requests:
#         cpu: "500m"
#         memory: "512Mi"
#       limits:
#         cpu: "2000m"
#         memory: "2Gi"
#   staging:
#     replicas: 2
#     resources:
#       requests:
#         cpu: "250m"
#         memory: "256Mi"

locals {
  env_config = local.app_config.environments[var.environment]
}

resource "kubernetes_deployment" "app" {
  spec {
    replicas = local.env_config.replicas

    template {
      spec {
        container {
          name  = "app"
          image = "app:${var.version}"

          resources {
            requests = local.env_config.resources.requests
            limits   = local.env_config.resources.limits
          }
        }
      }
    }
  }
}
```

## Step 2: Parse Helm Values from YAML

```hcl
# Load Helm values from environment-specific YAML files
locals {
  base_values = yamldecode(file("${path.module}/helm-values/base.yaml"))
  env_values  = yamldecode(file("${path.module}/helm-values/${var.environment}.yaml"))

  # Merge environment values over base values
  merged_values = merge(local.base_values, local.env_values)
}

resource "helm_release" "nginx" {
  name       = "nginx-ingress"
  repository = "https://kubernetes.github.io/ingress-nginx"
  chart      = "ingress-nginx"

  values = [yamlencode(local.merged_values)]
}
```

## Step 3: Kubernetes Manifests from YAML

```hcl
# Apply multiple Kubernetes manifests from YAML directory
locals {
  # Read all YAML files in a directory
  manifest_files = fileset("${path.module}/manifests", "*.yaml")

  manifests = {
    for f in local.manifest_files :
    trimsuffix(f, ".yaml") => yamldecode(file("${path.module}/manifests/${f}"))
  }
}

resource "kubernetes_manifest" "from_yaml" {
  for_each = local.manifests

  manifest = each.value
}
```

## Step 4: YAML to JSON Conversion

```hcl
# Convert YAML configs to JSON for APIs that require JSON
locals {
  yaml_policy = yamldecode(file("${path.module}/policies/iam-policy.yaml"))
  json_policy = jsonencode(local.yaml_policy)
}

resource "aws_iam_policy" "from_yaml" {
  name   = "yaml-policy"
  policy = local.json_policy
}

# Write parsed YAML as JSON output
output "config_as_json" {
  value = jsonencode(local.app_config)
}
```

## Summary

`yamldecode` in OpenTofu enables YAML-based configuration files, which many teams prefer over JSON due to comment support and cleaner syntax. Merging base and environment YAML files with `merge()` creates a layered configuration system where environment-specific values override defaults. The `yamlencode` function converts HCL maps back to YAML, enabling Helm values to be constructed programmatically and passed as YAML strings to `helm_release`.
