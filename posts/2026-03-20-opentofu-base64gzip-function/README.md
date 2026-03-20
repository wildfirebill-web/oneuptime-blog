---
title: "How to Use base64gzip() in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, functions, encoding
description: "Learn how to use the base64gzip() function in OpenTofu to compress and Base64-encode data."
---

# How to Use base64gzip() in OpenTofu

Learn how to use the base64gzip() function in OpenTofu to compress and Base64-encode data.

## Syntax

```hcl
# See OpenTofu documentation for full syntax
```

## Basic Example

```hcl
# Example usage in a configuration
locals {
  result = base64gzip(var.input)
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

The `base64gzip()` function is a useful tool in the OpenTofu expression language.
Refer to the [official OpenTofu documentation](https://opentofu.org/docs/language/functions/) for
the complete reference including all parameters and return values.
