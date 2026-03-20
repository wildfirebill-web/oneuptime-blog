# How to Use Provider-Defined Functions Introduced in OpenTofu 1.7

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Functions, OpenTofu 1.7, HCL, Infrastructure as Code

Description: Learn how to use provider-defined functions in OpenTofu 1.7 that extend HCL with custom functions provided by providers, such as AWS ARN parsing.

## Introduction

OpenTofu 1.7 introduced provider-defined functions, allowing providers to expose custom functions usable in HCL configurations. These functions are called with the `provider::<provider_name>::<function_name>()` syntax and enable richer data transformations without external data sources.

## Enabling and Using Provider Functions

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.20.0"  # provider functions require recent provider versions
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

## AWS Provider ARN Functions

The AWS provider exposes functions for parsing and constructing ARNs.

```hcl
# Parse an ARN into its components

locals {
  bucket_arn = "arn:aws:s3:::my-example-bucket"

  # provider::aws::arn_parse() returns an object with ARN components
  parsed_arn = provider::aws::arn_parse(local.bucket_arn)
}

output "arn_partition" {
  value = local.parsed_arn.partition  # "aws"
}

output "arn_service" {
  value = local.parsed_arn.service   # "s3"
}

output "arn_region" {
  value = local.parsed_arn.region    # "" (S3 ARNs have no region)
}

output "arn_account_id" {
  value = local.parsed_arn.account_id # ""
}

output "arn_resource" {
  value = local.parsed_arn.resource   # "my-example-bucket"
}
```

## Using Functions in Resource Configuration

```hcl
data "aws_caller_identity" "current" {}

locals {
  # Extract account ID from the caller's ARN
  caller_arn   = data.aws_caller_identity.current.arn
  parsed_caller = provider::aws::arn_parse(local.caller_arn)

  # Use the extracted account ID in resource names
  bucket_name = "my-app-${local.parsed_caller.account_id}"
}

resource "aws_s3_bucket" "app" {
  bucket = local.bucket_name
}
```

## Trimming ARN Suffixes

```hcl
# Some resources return ARNs with suffixes that need to be stripped
locals {
  role_arn = "arn:aws:iam::123456789012:role/MyRole/path"

  # Trim to just the role ARN without path
  parsed = provider::aws::arn_parse(local.role_arn)
  base_arn = "arn:${local.parsed.partition}:iam::${local.parsed.account_id}:role/${split("/", local.parsed.resource)[1]}"
}
```

## Writing Provider Functions (Provider Authors)

If you're writing a custom provider, you can define functions using the Plugin Framework:

```go
// In your provider implementation (Go)
func (p *ExampleProvider) Functions(_ context.Context) []func() function.Function {
    return []func() function.Function{
        NewParseResourceIdFunction,
    }
}

func NewParseResourceIdFunction() function.Function {
    return function.NewFunction(
        function.WithSummary("Parse a compound resource ID"),
        function.WithParameter(function.StringParameter{
            Name: "id",
        }),
        function.WithReturn(function.ObjectReturn{
            AttributeTypes: map[string]attr.Type{
                "org":  types.StringType,
                "name": types.StringType,
            },
        }),
        function.WithRunFunction(func(ctx context.Context, req function.RunRequest, resp *function.RunResponse) {
            var id string
            req.Arguments.Get(ctx, &id)
            parts := strings.SplitN(id, "/", 2)
            resp.Result.Set(ctx, map[string]string{
                "org":  parts[0],
                "name": parts[1],
            })
        }),
    )
}
```

## Checking Available Functions

```bash
# After init, provider functions are documented in the provider's registry page
# Or check via the provider's source code

# You can also test functions using the console
echo 'provider::aws::arn_parse("arn:aws:s3:::my-bucket")' | tofu console
```

## Summary

Provider-defined functions in OpenTofu 1.7 extend HCL with provider-specific operations like ARN parsing, reducing the need for complex string manipulation with `split()` and `regex()`. The `provider::<name>::<function>()` syntax makes provider functions discoverable and explicitly scoped. As more providers adopt the Plugin Framework, the ecosystem of available functions will grow.
