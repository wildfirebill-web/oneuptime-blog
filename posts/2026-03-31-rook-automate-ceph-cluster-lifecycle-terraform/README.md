# How to Automate Ceph Cluster Lifecycle with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Terraform, Lifecycle, Automation, Infrastructure as Code

Description: Automate the full Ceph cluster lifecycle including provisioning, scaling, upgrades, and decommissioning using Terraform with proper state management and safety checks.

---

Managing the full lifecycle of a Ceph cluster - from initial provisioning through scaling and eventual decommissioning - benefits from automation to prevent configuration drift and ensure consistency across environments.

## Lifecycle Phases

A complete Ceph cluster lifecycle managed by Terraform covers:
1. Bootstrap: VMs, network, Kubernetes
2. Install: Rook operator and CephCluster
3. Configure: pools, users, object stores
4. Scale: add nodes and OSDs
5. Upgrade: rolling Ceph version updates
6. Decommission: safe data migration and teardown

## Bootstrap Phase

```hcl
# main.tf - Bootstrap infrastructure
module "kubernetes_cluster" {
  source = "./modules/k8s"

  cluster_name    = var.cluster_name
  node_count      = var.storage_node_count
  node_flavor     = var.storage_node_flavor
  environment     = var.environment
}

module "rook_operator" {
  source = "./modules/rook-operator"

  namespace   = "rook-ceph"
  chart_version = var.rook_version

  depends_on = [module.kubernetes_cluster]
}
```

## Install Phase

```hcl
module "ceph_cluster" {
  source = "./modules/ceph-cluster"

  namespace     = "rook-ceph"
  ceph_version  = var.ceph_version
  mon_count     = var.mon_count
  osd_count     = var.storage_node_count
  replica_size  = var.replica_size

  depends_on = [module.rook_operator]
}
```

## Scaling with Count Variables

```hcl
# Scale by changing variables and running terraform apply
variable "storage_node_count" {
  description = "Number of storage nodes"
  type        = number
  default     = 3
}

# terraform.tfvars - update to scale
# storage_node_count = 6

resource "kubernetes_manifest" "ceph_cluster" {
  manifest = {
    # ...
    spec = {
      storage = {
        storageClassDeviceSets = [
          {
            name  = "set1"
            count = var.storage_node_count
            # ...
          }
        ]
      }
    }
  }
}
```

## Upgrade Workflow

Manage Ceph version upgrades through Terraform variables with controlled rollout:

```hcl
variable "ceph_version" {
  description = "Ceph container image version"
  type        = string
  default     = "v18.2.0"
  # Update to "v19.2.0" to trigger upgrade
}

resource "kubernetes_manifest" "ceph_cluster" {
  manifest = {
    spec = {
      cephVersion = {
        image = "quay.io/ceph/ceph:${var.ceph_version}"
        allowUnsupported = false
      }
    }
  }

  lifecycle {
    # Prevent accidental version downgrade
    precondition {
      condition     = !startswith(var.ceph_version, "v16")
      error_message = "Ceph versions below v17 are not supported."
    }
  }
}
```

## Pre-Upgrade Health Check

```hcl
resource "terraform_data" "pre_upgrade_check" {
  triggers_replace = { ceph_version = var.ceph_version }

  provisioner "local-exec" {
    command = <<-EOT
      echo "Checking cluster health before upgrade..."
      health=$(kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
        ceph health --format json | python3 -c \
        "import sys,json; print(json.load(sys.stdin)['status'])")

      if [[ "$health" != "HEALTH_OK" ]]; then
        echo "ERROR: Cluster not healthy ($health). Aborting upgrade."
        exit 1
      fi
      echo "Cluster healthy. Proceeding with upgrade to ${var.ceph_version}"
    EOT
  }
}
```

## Decommission Workflow

```hcl
# Set this to true to decommission
variable "decommission" {
  type    = bool
  default = false
}

# Lifecycle rule prevents accidental deletion in prod
resource "kubernetes_manifest" "ceph_cluster" {
  lifecycle {
    prevent_destroy = true
  }
  # ...
}

# Separate decommission module that removes prevent_destroy
module "decommission" {
  count  = var.decommission ? 1 : 0
  source = "./modules/decommission"

  namespace    = "rook-ceph"
  cluster_name = var.cluster_name
}
```

## Environment Promotion

```bash
# Promote config from staging to production
cd environments/staging
terraform output -json cluster_config > /tmp/staging-config.json

# Review and apply to prod
cd ../prod
# Update terraform.tfvars with tested configuration values
terraform plan
terraform apply
```

## Summary

Automating the full Ceph cluster lifecycle with Terraform enables consistent provisioning, controlled scaling, and safe upgrades through variable-driven workflows. Lifecycle rules, pre-upgrade health checks, and environment promotion patterns prevent accidental changes while keeping the entire cluster configuration version-controlled and auditable.
