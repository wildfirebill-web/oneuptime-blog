# How to Use base64decode() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Function, Encoding

Description: Learn how to use the base64decode() function in OpenTofu to decode Base64-encoded strings.

Learn how to use the base64decode() function in OpenTofu to decode Base64-encoded strings.

## Syntax

```hcl
# See OpenTofu documentation for full syntax

```

## Basic Example

```hcl
# Example usage in a configuration
locals {
  result = base64decode(var.input)
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

The `base64decode()` function is a useful tool in the OpenTofu expression language.
Refer to the [official OpenTofu documentation](https://opentofu.org/docs/language/functions/) for
the complete reference including all parameters and return values.
