# How to Use the enabled Meta-Argument with Data Sources in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Data Source, Enabled, Meta-Arguments, Infrastructure as Code, DevOps

Description: A guide to using the enabled meta-argument with data sources in OpenTofu to conditionally fetch data based on configuration variables.

## Introduction

The `enabled` meta-argument works with data sources just as it does with resources. When `enabled = false`, the data source is not queried during plan and apply operations, and its attributes return `null`. This is useful for conditionally fetching data only when certain features are active.

## Basic enabled with Data Sources

```hcl
variable "use_existing_vpc" {
  type    = bool
  default = false
}

# Only query existing VPC if we're using one

data "aws_vpc" "existing" {
  enabled = var.use_existing_vpc
  id      = var.existing_vpc_id
}

# Create new VPC when not using existing
resource "aws_vpc" "new" {
  enabled    = !var.use_existing_vpc
  cidr_block = var.vpc_cidr
}

locals {
  vpc_id = var.use_existing_vpc ? data.aws_vpc.existing.id : aws_vpc.new.id
}
```

## Conditional Secret Fetching

```hcl
variable "use_secrets_manager" {
  type    = bool
  default = true
}

variable "db_password_direct" {
  type      = string
  default   = null
  sensitive = true
}

# Only fetch from Secrets Manager if using it
data "aws_secretsmanager_secret_version" "db_password" {
  enabled   = var.use_secrets_manager
  secret_id = "myapp/db-password"
}

locals {
  db_password = var.use_secrets_manager ? (
    jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string).password
  ) : var.db_password_direct
}
```

## Feature-Gated Data Sources

```hcl
variable "features" {
  type = object({
    enable_waf       = bool
    enable_guardduty = bool
    enable_sso       = bool
  })
}

# Only fetch WAF ACL if WAF feature is enabled
data "aws_wafv2_web_acl" "existing" {
  enabled = var.features.enable_waf
  name    = "shared-web-acl"
  scope   = "REGIONAL"
}

# Only fetch GuardDuty detector if feature is enabled
data "aws_guardduty_detector" "main" {
  enabled = var.features.enable_guardduty
}

resource "aws_wafv2_web_acl_association" "app" {
  enabled      = var.features.enable_waf
  resource_arn = aws_lb.app.arn
  web_acl_arn  = data.aws_wafv2_web_acl.existing.arn
}
```

## Environment-Specific Data Sources

```hcl
variable "environment" {
  type = string
}

# Only look up existing ACM certificate in production
# (dev/staging use self-signed or Let's Encrypt)
data "aws_acm_certificate" "app" {
  enabled = var.environment == "prod"
  domain  = "app.example.com"
  statuses = ["ISSUED"]
}

# Only query existing Route53 zone in non-local environments
data "aws_route53_zone" "main" {
  enabled = contains(["staging", "prod"], var.environment)
  name    = "example.com"
}
```

## Using enabled to Avoid Errors

```hcl
variable "cluster_exists" {
  type    = bool
  default = false
}

variable "cluster_name" {
  type    = string
  default = ""
}

# Avoid querying a cluster that doesn't exist yet
data "aws_eks_cluster" "existing" {
  enabled = var.cluster_exists && var.cluster_name != ""
  name    = var.cluster_name != "" ? var.cluster_name : "placeholder"
}

data "aws_eks_cluster_auth" "existing" {
  enabled = var.cluster_exists && var.cluster_name != ""
  name    = var.cluster_name != "" ? var.cluster_name : "placeholder"
}
```

## Handling null Values from Disabled Data Sources

```hcl
variable "use_custom_kms_key" {
  type    = bool
  default = false
}

data "aws_kms_key" "custom" {
  enabled  = var.use_custom_kms_key
  key_id   = var.kms_key_alias
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.use_custom_kms_key ? "aws:kms" : "AES256"
      kms_master_key_id = var.use_custom_kms_key ? data.aws_kms_key.custom.arn : null
    }
  }
}
```

## Multiple Conditional Data Sources

```hcl
variable "storage_backend" {
  type    = string
  default = "s3"
  # Options: "s3", "gcs", "azure"
}

data "aws_s3_bucket" "storage" {
  enabled = var.storage_backend == "s3"
  bucket  = var.s3_bucket_name
}

data "google_storage_bucket" "storage" {
  enabled = var.storage_backend == "gcs"
  name    = var.gcs_bucket_name
}

locals {
  storage_endpoint = (
    var.storage_backend == "s3" ? "https://${data.aws_s3_bucket.storage.bucket_regional_domain_name}" :
    var.storage_backend == "gcs" ? "https://storage.googleapis.com/${data.google_storage_bucket.storage.name}" :
    var.azure_storage_endpoint
  )
}
```

## Conclusion

The `enabled` meta-argument on data sources prevents unnecessary queries to cloud APIs when the data isn't needed. This is particularly useful for multi-environment configurations where some infrastructure only exists in certain environments, feature flags that control which external services are used, and avoiding errors when querying resources that may not exist yet. When a data source has `enabled = false`, all its attributes return `null`, so use null coalescing or conditional expressions when consuming the data.
