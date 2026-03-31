# How to Fix 'Error: Module Not Found' in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Module, Error, Infrastructure as Code, Debugging

Description: Learn how to diagnose and fix module not found errors in OpenTofu caused by incorrect source paths, missing tofu init, and registry connectivity issues.

## Introduction

"Module not found" or "Module not installed" errors occur when OpenTofu cannot locate a module at the path or registry address specified in the `source` argument. The fix depends on whether the module is local, from a Git repository, or from a registry.

## Common Error Messages

```hcl
Error: Module not installed
  on main.tf line 10, in module "vpc":
  10:   source = "./modules/vpc"
  This module is not yet installed. Run "tofu init" to install all modules required by this configuration.

Error: Failed to download module
  Could not download module "github.com/my-org/modules//vpc?ref=v1.2.0" (main.tf:10) source code
  from "git::https://github.com/my-org/modules.git//vpc?ref=v1.2.0": error downloading
  'https://github.com/my-org/modules/archive/v1.2.0.zip': 404 Not Found
```

## Fix 1: Run tofu init

Most "module not installed" errors require simply running init:

```bash
# Download and install all modules

tofu init

# If you added a new module source, init again
tofu init -upgrade
```

## Fix 2: Fix Local Module Path

Local paths are relative to the module file containing the `source` argument:

```hcl
# WRONG - path doesn't match directory structure
module "vpc" {
  source = "./module/vpc"   # Directory is actually ./modules/vpc
}

# CORRECT
module "vpc" {
  source = "./modules/vpc"
}
```

```bash
# Verify the module directory exists
ls -la ./modules/vpc/
# Should contain: main.tf, variables.tf, outputs.tf
```

## Fix 3: Fix Git Module Source

```hcl
# WRONG - tag doesn't exist
module "vpc" {
  source = "git::https://github.com/my-org/tf-modules.git//modules/vpc?ref=v1.2.0"
}

# CORRECT - verify the ref exists
# Check: git ls-remote https://github.com/my-org/tf-modules.git | grep refs/tags/v1.2.0
module "vpc" {
  source = "git::https://github.com/my-org/tf-modules.git//modules/vpc?ref=v1.3.0"
}
```

For private repositories, configure Git credentials:

```bash
# Configure git to use your token
git config --global url."https://oauth2:${GITHUB_TOKEN}@github.com".insteadOf "https://github.com"
tofu init
```

## Fix 4: Fix Registry Module Source

```hcl
# WRONG - module namespace/name is incorrect
module "vpc" {
  source  = "terraform-aws-modules/aws-vpc/aws"  # Wrong
  version = "5.0"
}

# CORRECT - format is <namespace>/<module>/<provider>
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

```bash
# Test registry connectivity
curl -s https://registry.opentofu.org/v1/modules/terraform-aws-modules/vpc/aws/versions | jq '.modules[0].versions[0]'
```

## Fix 5: Submodule Double-Slash Separator

For Git and registry sources that include a subdirectory, use `//` (double slash):

```hcl
# WRONG - single slash won't navigate into the subdirectory
module "vpc" {
  source = "git::https://github.com/my-org/tf-modules.git/modules/vpc"
}

# CORRECT - double slash separates repo root from subdirectory path
module "vpc" {
  source = "git::https://github.com/my-org/tf-modules.git//modules/vpc?ref=main"
}
```

## Conclusion

Module not found errors are resolved by running `tofu init`, fixing the source path (local, Git, or registry), ensuring the tag/ref exists, and configuring authentication for private repositories. Use the double-slash separator for Git subdirectories and validate registry module names against the public registry search.
