# How to Use Terraform Rancher2 Provider

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Terraform, Infrastructure as Code, Rancher2 Provider

Description: Deep dive into the Terraform Rancher2 provider features including data sources, resource imports, and managing complex Rancher configurations with Terraform.

## Introduction

The Terraform Rancher2 provider is the official way to manage Rancher resources programmatically. This guide provides a deep dive into the provider's capabilities, best practices for state management, and advanced patterns for managing complex Rancher environments with modules.

## Prerequisites

- Terraform 1.5+ installed
- Rancher 2.7+ instance
- API token with appropriate permissions

## Step 1: Provider Authentication Methods

```hcl
# provider-methods.tf - Different authentication approaches
# Method 1: API Token (recommended for CI/CD)
provider "rancher2" {
  api_url   = "https://rancher.example.com"
  token_key = "token-xxxxx:yyyyyyyyyy"
}

# Method 2: Username/Password (not recommended for production)
provider "rancher2" {
  api_url   = "https://rancher.example.com"
  access_key = "token-xxxxx"
  secret_key = "yyyyyyyyyy"
}

# Method 3: Bootstrapping (first-time setup)
provider "rancher2" {
  api_url  = "https://rancher.example.com"
  bootstrap = true
}

resource "rancher2_bootstrap" "admin" {
  password  = var.admin_password
  telemetry = false
}
```

## Step 2: Data Sources

```hcl
# data-sources.tf - Read existing Rancher resources
# Get existing cluster details
data "rancher2_cluster" "existing_cluster" {
  name = "existing-production"
}

# Get existing project
data "rancher2_project" "system_project" {
  cluster_id = data.rancher2_cluster.existing_cluster.id
  name       = "System"
}

# Get existing namespace
data "rancher2_namespace" "cattle_system" {
  name       = "cattle-system"
  project_id = data.rancher2_project.system_project.id
}

# Get catalog
data "rancher2_catalog_v2" "rancher_charts" {
  cluster_id = data.rancher2_cluster.existing_cluster.id
  name       = "rancher-charts"
}

# Use data source outputs
output "cluster_id" {
  value = data.rancher2_cluster.existing_cluster.id
}

output "kubeconfig" {
  value     = data.rancher2_cluster.existing_cluster.kube_config
  sensitive = true
}
```

## Step 3: Import Existing Resources

```bash
# Import existing resources into Terraform state
# First, find the resource ID in Rancher UI or API

# Import a cluster
terraform import rancher2_cluster_v2.imported_cluster local.c-12345

# Import a project
terraform import rancher2_project.imported_project c-12345:p-abcdef

# Import a namespace
terraform import rancher2_namespace.imported_ns p-abcdef:my-namespace

# After import, generate the Terraform config
terraform show -json > imported.json
```

## Step 4: Advanced Module Structure

```hcl
# modules/rancher-cluster/main.tf - Reusable cluster module
variable "cluster_name" {
  description = "Name of the cluster"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "v1.29.4+rke2r1"
}

variable "node_count" {
  description = "Number of worker nodes"
  type        = number
  default     = 3
}

variable "cloud_credential_name" {
  description = "Cloud credential name"
  type        = string
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}

resource "rancher2_cluster_v2" "cluster" {
  name               = var.cluster_name
  kubernetes_version = var.kubernetes_version

  rke_config {
    machine_pools {
      name                         = "control-plane"
      cloud_credential_secret_name = var.cloud_credential_name
      control_plane_role           = true
      etcd_role                    = true
      quantity                     = 3
      machine_config {
        kind = rancher2_machine_config_v2.cp_config.kind
        name = rancher2_machine_config_v2.cp_config.name
      }
    }

    machine_pools {
      name                         = "workers"
      cloud_credential_secret_name = var.cloud_credential_name
      worker_role                  = true
      quantity                     = var.node_count
      machine_config {
        kind = rancher2_machine_config_v2.worker_config.kind
        name = rancher2_machine_config_v2.worker_config.name
      }
    }
  }

  labels = merge(var.tags, {
    managed-by = "terraform"
  })
}

output "cluster_id" {
  value = rancher2_cluster_v2.cluster.cluster_v1_id
}

output "kubeconfig" {
  value     = rancher2_cluster_v2.cluster.kube_config
  sensitive = true
}
```

```hcl
# environments/production/main.tf - Use the module
module "production_cluster" {
  source = "../../modules/rancher-cluster"

  cluster_name          = "production"
  kubernetes_version    = "v1.29.4+rke2r1"
  node_count            = 5
  cloud_credential_name = "aws-production"

  tags = {
    environment = "production"
    team        = "platform"
    cost-center = "engineering"
  }
}

module "staging_cluster" {
  source = "../../modules/rancher-cluster"

  cluster_name          = "staging"
  kubernetes_version    = "v1.29.4+rke2r1"
  node_count            = 2
  cloud_credential_name = "aws-staging"

  tags = {
    environment = "staging"
    team        = "platform"
  }
}
```

## Step 5: Manage Cloud Credentials

```hcl
# credentials.tf - Cloud provider credentials
resource "rancher2_cloud_credential" "aws_prod" {
  name = "aws-production"
  amazonec2_credential_config {
    access_key = var.aws_access_key
    secret_key = var.aws_secret_key
  }
}

resource "rancher2_cloud_credential" "azure_prod" {
  name = "azure-production"
  azure_credential_config {
    client_id       = var.azure_client_id
    client_secret   = var.azure_client_secret
    subscription_id = var.azure_subscription_id
  }
}
```

## Step 6: Configure Node Templates

```hcl
# node-template.tf - Node template for custom node pools
resource "rancher2_node_template" "aws_worker" {
  name            = "aws-worker-t3xlarge"
  cloud_credential_id = rancher2_cloud_credential.aws_prod.id
  engine_insecure_registry = ["registry.internal.example.com"]

  amazonec2_config {
    ami            = "ami-0c02fb55956c7d316"
    region         = "us-east-1"
    security_group = ["rancher-workers"]
    subnet_id      = var.worker_subnet_id
    vpc_id         = var.vpc_id
    zone           = "a"
    instance_type  = "t3.xlarge"
    root_size      = "100"
    tags           = "environment,production"

    # IAM role for ECR access
    iam_instance_profile = "EC2-ECR-ReadOnly"
  }
}
```

## Step 7: Handle State and Workspaces

```bash
# Use Terraform workspaces for environment isolation
terraform workspace new production
terraform workspace new staging

# Switch workspace
terraform workspace select production
terraform apply

terraform workspace select staging
terraform apply

# List workspaces
terraform workspace list

# State management
terraform state list          # List all resources
terraform state show rancher2_cluster_v2.production_cluster
terraform state rm rancher2_user.old_user  # Remove from state without destroying
terraform state mv rancher2_cluster_v2.old rancher2_cluster_v2.new  # Rename
```

## Step 8: CI/CD Pipeline Integration

```yaml
# .github/workflows/terraform.yml - GitHub Actions for Terraform
name: Terraform Rancher

on:
  push:
    branches: [main]
    paths: ['terraform/**']
  pull_request:
    paths: ['terraform/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Terraform Init
        run: terraform init
        working-directory: terraform/environments/production
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Plan
        run: terraform plan -no-color
        working-directory: terraform/environments/production
        env:
          TF_VAR_rancher_url: ${{ secrets.RANCHER_URL }}
          TF_VAR_rancher_token: ${{ secrets.RANCHER_TOKEN }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
        working-directory: terraform/environments/production
        env:
          TF_VAR_rancher_url: ${{ secrets.RANCHER_URL }}
          TF_VAR_rancher_token: ${{ secrets.RANCHER_TOKEN }}
```

## Conclusion

The Terraform Rancher2 provider offers comprehensive coverage of the Rancher API, enabling full infrastructure-as-code management. By using modules for reusable patterns, workspaces for environment isolation, and CI/CD pipelines for automated changes, you can build a robust, version-controlled Rancher management system. Always pin provider versions, use remote state for collaboration, and implement proper secret management with vault or similar tools for credentials.
