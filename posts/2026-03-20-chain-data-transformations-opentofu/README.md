# How to Chain Data Transformations in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HCL, Data Transformation, Functions, Infrastructure as Code, Locals

Description: Learn how to chain multiple OpenTofu functions and local values to perform complex data transformations on infrastructure inputs and outputs.

---

OpenTofu's HCL supports a rich set of built-in functions and local values that you can chain together to transform data — parsing strings, reshaping maps, filtering lists, and generating structured outputs — without external scripting.

---

## Basic Chaining with Locals

```hcl
locals {
  # Raw input
  raw_tags = "env=prod,team=platform,owner=ops"

  # Step 1: Split into key=value pairs
  tag_pairs = split(",", local.raw_tags)

  # Step 2: Convert to a map
  tag_map = {
    for pair in local.tag_pairs :
    split("=", pair)[0] => split("=", pair)[1]
  }
}
# Result: { env = "prod", team = "platform", owner = "ops" }
```

---

## Filter and Transform a List

```hcl
locals {
  all_regions = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]

  # Filter to US regions only
  us_regions = [for r in local.all_regions : r if startswith(r, "us-")]

  # Transform to uppercase
  us_regions_upper = [for r in local.us_regions : upper(r)]
}
```

---

## Build a Nested Map from a Flat List

```hcl
locals {
  subnets = [
    { name = "web-a", cidr = "10.0.1.0/24", az = "us-east-1a" },
    { name = "web-b", cidr = "10.0.2.0/24", az = "us-east-1b" },
    { name = "db-a",  cidr = "10.0.3.0/24", az = "us-east-1a" },
  ]

  # Group subnets by AZ
  subnets_by_az = {
    for s in local.subnets :
    s.az => s...
  }
}
```

---

## Merge Maps with Override

```hcl
locals {
  defaults = {
    instance_type = "t3.micro"
    ami           = "ami-0c55b159cbfafe1f0"
    monitoring    = false
  }

  overrides = {
    instance_type = "t3.medium"
    monitoring    = true
  }

  final_config = merge(local.defaults, local.overrides)
  # instance_type = "t3.medium", ami = "ami-0c55...", monitoring = true
}
```

---

## String Transformation Pipeline

```hcl
locals {
  raw_name    = "  My App Server  "
  clean_name  = trimspace(local.raw_name)         # "My App Server"
  lower_name  = lower(local.clean_name)            # "my app server"
  slug        = replace(local.lower_name, " ", "-") # "my-app-server"
  resource_id = "ec2-${local.slug}"                # "ec2-my-app-server"
}
```

---

## Summary

Chain OpenTofu transformations using `locals` blocks with built-in functions like `split`, `merge`, `for` expressions, `replace`, `trimspace`, and `startswith`. Each local value can reference previous locals, creating a readable pipeline. This approach centralizes complex logic in one place, making your resource blocks clean and your transformations testable with `tofu console`.
