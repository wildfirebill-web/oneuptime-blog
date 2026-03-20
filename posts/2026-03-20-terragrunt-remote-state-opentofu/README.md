# How to Configure Terragrunt Remote State for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terragrunt, Remote State, S3 Backend, Backend Configuration

Description: Learn how to configure Terragrunt's remote_state block to automatically generate and manage OpenTofu backend configurations across all modules without repeating backend settings.

## Introduction

Terragrunt's `remote_state` block generates the OpenTofu backend configuration for every module, using the module's path as part of the state key. This ensures each module gets its own state file with a predictable, consistent naming convention.

## Root remote_state Configuration

```hcl
# Root terragrunt.hcl

locals {
  env_vars   = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env        = local.env_vars.locals.environment
  region     = local.env_vars.locals.aws_region
  account_id = local.env_vars.locals.aws_account_id
}

remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket  = "my-company-tofu-state-${local.account_id}"
    # path_relative_to_include() returns the relative path from root
    # e.g., "environments/prod/networking"
    key     = "${path_relative_to_include()}/terraform.tfstate"
    region  = local.region
    encrypt = true

    dynamodb_table = "tofu-state-lock"

    # Enable S3 versioning for state backup
    enable_lock_table_ssencryption = true
    s3_bucket_tags = {
      ManagedBy = "Terragrunt"
    }
  }
}
```

## Auto-Creating the Backend S3 Bucket

Terragrunt can create the S3 bucket and DynamoDB table automatically:

```hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "my-company-tofu-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.region
    encrypt        = true
    dynamodb_table = "tofu-state-lock"

    # Terragrunt will create these if they don't exist
    s3_bucket_tags = {
      Purpose   = "OpenTofu State"
      ManagedBy = "Terragrunt"
    }
    dynamodb_table_tags = {
      Purpose   = "OpenTofu State Lock"
      ManagedBy = "Terragrunt"
    }
  }
}
```

## Workspace-Based State Keys

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket    = "my-company-tofu-state"
    # Include workspace in state key for workspace-based deployments
    key       = "${local.env}/${path_relative_to_include()}/terraform.tfstate"
    region    = local.region
    encrypt   = true
    dynamodb_table = "tofu-state-lock"
  }
}
```

## GCS Backend for Google Cloud

```hcl
remote_state {
  backend = "gcs"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket = "my-company-tofu-state-${local.project_id}"
    prefix = path_relative_to_include()
  }
}
```

## Reading Outputs from Remote State

```hcl
# Use terraform_remote_state to read other modules' outputs
# Terragrunt makes this easy with dependency blocks (preferred)
# but you can also use direct remote state references:

dependency "networking" {
  config_path = "../networking"
}

# Alternatively, for cross-repo dependencies:
data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = "my-company-tofu-state"
    key    = "shared/networking/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Verifying Remote State Configuration

```bash
# Verify the generated backend.tf looks correct
cd environments/prod/networking
terragrunt init --terragrunt-log-level debug 2>&1 | grep -A 10 "Backend config"

# Check the generated backend.tf file
cat .terraform/environment
cat backend.tf
```

## Conclusion

Terragrunt's `remote_state` block eliminates the most painful part of multi-module OpenTofu setups - managing backend configuration in every directory. The auto-generated `backend.tf` with a path-based key creates an intuitive state file hierarchy that mirrors your directory structure, making it easy to find and inspect any module's state.
