# How to Use the sha256() Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Functions, Crypto

Description: Learn how to use the sha256() function in OpenTofu to compute SHA-256 hashes for data integrity.

Learn how to use the sha256() function in OpenTofu to compute SHA-256 hashes for data integrity.

## Syntax

```hcl
# See OpenTofu documentation for full syntax

```

## Basic Example

```hcl
# Example usage in a configuration
locals {
  result = sha256(var.input)
}

output "result" {
  value = local.result
}
```

## Practical Use Case

This function is part of OpenTofu's built-in function library and can be used
in any expression within your configuration.

## Related Functions

See the related function guides in this OpenTofu blog series for more examples.

## Conclusion

The `sha256()` function is a useful tool in the OpenTofu expression language.
Refer to the [official OpenTofu documentation](https://opentofu.org/docs/language/functions/) for
the complete reference including all parameters and return values.
