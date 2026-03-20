---
title: "Using HTTP URLs as Module Sources in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, modules, http
description: "Learn how to use HTTP and HTTPS URLs as module sources in OpenTofu to download modules from web servers."
---

# Using HTTP URLs as Module Sources in OpenTofu

OpenTofu can download module packages from HTTP and HTTPS URLs. The URL must point to a zip, tar.gz, tar.bz2, or tar.xz archive.

## Basic HTTPS URL

```hcl
module "vpc" {
  source = "https://example.com/modules/vpc.zip"

  cidr_block  = "10.0.0.0/16"
  environment = "production"
}
```

## With Checksums

For security, you can pin to a specific file hash:

```hcl
module "vpc" {
  source = "https://releases.mycompany.com/modules/vpc-2.1.0.tar.gz"
  # OpenTofu verifies the checksum if you provide it
}
```

## Using Archive Files from S3 Pre-signed URLs

```hcl
# Generate pre-signed URL in advance
# aws s3 presign s3://my-modules/vpc-2.1.0.zip --expires-in 3600

module "vpc" {
  source = "https://my-modules.s3.amazonaws.com/vpc-2.1.0.zip?X-Amz-Algorithm=..."
}
```

## Self-Hosted Module Server

```hcl
# Internal HTTP server serving module archives
module "database" {
  source = "https://modules.internal.example.com/database/v1.5.0.tar.gz"

  engine    = "postgres"
  version   = "14"
  vpc_id    = var.vpc_id
  subnet_ids = var.subnet_ids
}
```

## Authenticating with HTTP Sources

```bash
# Set credentials in environment
export TF_HTTP_USERNAME="myuser"
export TF_HTTP_PASSWORD="mypassword"

# Or use netrc for more control
echo "machine modules.example.com login myuser password mypassword" >> ~/.netrc
chmod 600 ~/.netrc
```

## Practical: CI/CD with Pre-built Module Archives

```bash
# CI pipeline: build and publish module archive
tar -czf vpc-module-v${VERSION}.tar.gz ./modules/vpc/
aws s3 cp vpc-module-v${VERSION}.tar.gz s3://my-modules/
```

```hcl
# OpenTofu configuration
variable "vpc_module_version" {
  default = "1.5.0"
}

module "vpc" {
  source = "https://my-modules.s3.amazonaws.com/vpc-module-v${var.vpc_module_version}.tar.gz"

  cidr_block  = "10.0.0.0/16"
  environment = var.environment
}
```

## Conclusion

HTTP URL sources are flexible for organizations with existing artifact repositories or web servers. They work with any HTTP server that can serve files. For production use, always pin to specific versioned archives, use HTTPS for security, and consider pairing with checksums for integrity verification.
