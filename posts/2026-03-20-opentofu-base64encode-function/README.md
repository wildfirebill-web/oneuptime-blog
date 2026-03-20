# How to Use base64encode() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Functions, Encoding

Description: Learn how to use the base64encode() function in OpenTofu to encode strings as Base64.

Learn how to use the base64encode() function in OpenTofu to encode strings as Base64.

## Syntax

```hcl
# See OpenTofu documentation for full syntax

```

## Basic Example

```hcl
# Example usage in a configuration
locals {
  result = base64encode(var.input)
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

The `base64encode()` function is a useful tool in the OpenTofu expression language.
Refer to the [official OpenTofu documentation](https://opentofu.org/docs/language/functions/) for
the complete reference including all parameters and return values.
