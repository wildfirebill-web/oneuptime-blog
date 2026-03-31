# How to Use replace() and split() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Function, String

Description: Learn how to use the replace() and split() functions in OpenTofu to manipulate strings and create lists.

`replace()` substitutes occurrences of a substring in a string, while `split()` divides a string into a list of substrings at a given delimiter. Together they're powerful for string parsing and normalization.

## replace()

Replaces all occurrences of a substring with another:

```hcl
> replace("hello world", "world", "OpenTofu")
"hello OpenTofu"

> replace("us-east-1", "-", "_")
"us_east_1"
```

Supports regex when the search string is wrapped in `/`:

```hcl
> replace("hello 123 world 456", "/[0-9]+/", "#")
"hello # world #"
```

```hcl
variable "project_name" {
  type    = string
  default = "My Awesome Project"
}

locals {
  # Create URL-safe slug
  slug = lower(replace(var.project_name, " ", "-"))
  # "my-awesome-project"
  
  # Create valid S3 bucket name  
  bucket_name = replace(lower(var.project_name), " ", "-")
}
```

## split()

Divides a string into a list using a delimiter:

```hcl
> split(",", "us-east-1,us-west-2,eu-west-1")
["us-east-1", "us-west-2", "eu-west-1"]

> split("/", "path/to/my/file")
["path", "to", "my", "file"]
```

```hcl
variable "allowed_cidrs" {
  description = "Comma-separated list of allowed CIDRs"
  type        = string
  default     = "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
}

resource "aws_security_group_rule" "allow_internal" {
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = split(",", var.allowed_cidrs)
  
  security_group_id = aws_security_group.app.id
}
```

## Parsing ARNs with split()

```hcl
locals {
  # Parse parts of an ARN
  role_arn  = "arn:aws:iam::123456789012:role/my-role"
  arn_parts = split(":", local.role_arn)
  account_id = local.arn_parts[4]  # "123456789012"
  
  # Get the role name from the last part
  role_name  = split("/", local.arn_parts[5])[1]  # "my-role"
}
```

## Combining replace() and split()

```hcl
variable "csv_tags" {
  description = "Tags in key=value,key=value format"
  type        = string
  default     = "env=prod,team=platform,cost-center=123"
}

locals {
  tag_pairs = split(",", var.csv_tags)
  # ["env=prod", "team=platform", "cost-center=123"]
  
  tags = {
    for pair in local.tag_pairs :
    split("=", pair)[0] => split("=", pair)[1]
  }
  # { env = "prod", team = "platform", cost-center = "123" }
}
```

## Conclusion

`replace()` and `split()` are essential for string manipulation in OpenTofu. Use `replace()` to normalize strings (spaces to hyphens, uppercase to lowercase), and `split()` to parse delimited strings into lists. The combination enables powerful data transformation patterns, especially for processing user input and CSV-format variable values.
