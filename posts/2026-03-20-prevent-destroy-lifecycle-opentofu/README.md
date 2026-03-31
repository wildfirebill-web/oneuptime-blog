# How to Use prevent_destroy Lifecycle in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resource, Lifecycle, Protect_destroy, Infrastructure as Code, DevOps

Description: A guide to using prevent_destroy lifecycle in OpenTofu to protect critical resources from accidental deletion.

## Introduction

The `prevent_destroy` lifecycle setting creates a safety guard against accidentally deleting critical resources. When set to `true`, any plan that would destroy the resource will fail with an error, forcing you to explicitly remove the protection before deletion is allowed.

## Basic prevent_destroy

```hcl
resource "aws_rds_cluster" "production" {
  cluster_identifier = "production-db"
  engine             = "aurora-postgresql"
  master_username    = "admin"
  master_password    = var.db_password

  lifecycle {
    prevent_destroy = true  # Cannot be destroyed while this is set
  }
}
```

## What Happens When You Try to Destroy

```bash
# Attempt to destroy:

tofu destroy

# Error:
# Error: Instance cannot be destroyed
#
#   on main.tf line 10:
#   10:   lifecycle {
#   11:     prevent_destroy = true
#   12:   }
#
# Resource aws_rds_cluster.production has lifecycle.prevent_destroy set,
# but the plan calls for this resource to be destroyed. To avoid this error
# and continue with the destruction, either remove the resource from the
# Terraform configuration or remove the lifecycle.prevent_destroy block
# from the resource configuration.
```

Resources to Protect

```hcl
# Database cluster
resource "aws_rds_cluster" "main" {
  cluster_identifier = "production-db"
  # ...

  lifecycle {
    prevent_destroy = true
  }
}

# S3 buckets with data
resource "aws_s3_bucket" "data" {
  bucket = "production-app-data"

  lifecycle {
    prevent_destroy = true
  }
}

# KMS keys
resource "aws_kms_key" "encryption" {
  description = "Production data encryption key"

  lifecycle {
    prevent_destroy = true
  }
}

# EKS cluster
resource "aws_eks_cluster" "production" {
  name = "production-cluster"

  lifecycle {
    prevent_destroy = true
  }
}
```

## Environment-Conditional Protection

```hcl
variable "environment" {
  type = string
}

resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-db"
  engine         = "postgres"
  instance_class = "db.t3.micro"

  lifecycle {
    # Only prevent_destroy in production
    prevent_destroy = var.environment == "prod"
  }
}
```

## Removing prevent_destroy for Intentional Deletion

```bash
# To intentionally delete a protected resource:

# Step 1: Remove or set prevent_destroy = false
# Edit main.tf:
# lifecycle {
#   prevent_destroy = false  # or remove the lifecycle block
# }

# Step 2: Apply the configuration change
tofu apply  # This updates the state, not the resource

# Step 3: Now destroy is allowed
tofu destroy -target=aws_rds_cluster.production

# Step 4: Re-add protect_destroy after deletion (for remaining resources)
```

## Combining with Other Lifecycle Settings

```hcl
resource "aws_rds_cluster" "production" {
  cluster_identifier = "production"
  engine             = "aurora-postgresql"

  lifecycle {
    prevent_destroy       = true   # Cannot be deleted
    create_before_destroy = true   # Zero-downtime if replaced
    ignore_changes        = [master_password]  # Ignore manual password changes
  }
}
```

## prevent_destroy in Modules

```hcl
# The lifecycle block must be in the module definition
# You cannot set it from outside the module

# In modules/rds/main.tf:
resource "aws_db_instance" "this" {
  # ...

  lifecycle {
    prevent_destroy = var.prevent_destroy
  }
}

# In modules/rds/variables.tf:
variable "prevent_destroy" {
  type    = bool
  default = false
}

# In root module, pass it to the child:
module "db" {
  source = "./modules/rds"

  prevent_destroy = var.environment == "prod"
}
```

## Conclusion

`prevent_destroy` is a critical safety feature for production infrastructure. Apply it to databases, encryption keys, DNS zones, and any resource where accidental deletion would cause severe data loss or service disruption. Use conditional expressions based on environment to automatically enable it in production without affecting development flexibility. Remember that removing the guard requires a code change and apply, creating a natural "break glass" procedure for intentional deletions.
