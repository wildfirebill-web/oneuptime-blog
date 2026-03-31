# How to Use strcontains(), startswith(), and endswith() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Function, String

Description: Learn how to use strcontains, startswith, and endswith functions in OpenTofu for string checking.

These three functions test whether a string contains a specific substring, begins with a specific prefix, or ends with a specific suffix. They return boolean values, making them ideal for use in conditions and validations.

## strcontains()

Returns `true` if a string contains a given substring:

```hcl
> strcontains("hello world", "world")
true

> strcontains("hello world", "xyz")
false
```

```hcl
variable "instance_type" {
  type = string
}

locals {
  is_graviton = strcontains(var.instance_type, "g.")  
  # true for "m6g.large", "c7g.xlarge", etc.
}

resource "aws_instance" "app" {
  ami           = local.is_graviton ? data.aws_ami.arm64.id : data.aws_ami.x86.id
  instance_type = var.instance_type
}
```

## startswith()

Returns `true` if a string begins with a given prefix:

```hcl
> startswith("hello world", "hello")
true

> startswith("hello world", "world")
false
```

```hcl
variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string

  validation {
    condition     = startswith(var.ami_id, "ami-")
    error_message = "AMI ID must start with 'ami-'."
  }
}
```

## endswith()

Returns `true` if a string ends with a given suffix:

```hcl
> endswith("main.tf", ".tf")
true

> endswith("main.tf", ".py")
false
```

```hcl
variable "bucket_name" {
  type = string
  
  validation {
    condition     = !endswith(var.bucket_name, "-")
    error_message = "Bucket name must not end with a hyphen."
  }
}
```

## Combining in Conditions

```hcl
variable "region" {
  type = string
}

locals {
  is_us_region    = startswith(var.region, "us-")
  is_eu_region    = startswith(var.region, "eu-")
  is_gov_region   = strcontains(var.region, "gov")
  is_east_region  = endswith(var.region, "-east-1")
}

resource "aws_s3_bucket" "data" {
  bucket = "myapp-data-${var.region}"
  
  tags = {
    DataResidency = local.is_eu_region ? "EU" : (local.is_us_region ? "US" : "OTHER")
    GovCloud      = local.is_gov_region ? "true" : "false"
  }
}
```

## Filtering Lists with These Functions

```hcl
variable "instance_types" {
  type    = list(string)
  default = ["t3.micro", "m5.large", "c5.xlarge", "t4g.micro", "m6g.large"]
}

locals {
  # Filter to only graviton instances
  graviton_types = [for t in var.instance_types : t if endswith(t, "g.micro") || strcontains(t, "g.")]
  # ["t4g.micro", "m6g.large"]
  
  # Filter to only T-series instances
  t_series = [for t in var.instance_types : t if startswith(t, "t")]
  # ["t3.micro", "t4g.micro"]
}
```

## Conclusion

`strcontains()`, `startswith()`, and `endswith()` make it easy to write conditional logic based on string content without needing regex. Use them for input validation, conditional resource configuration, and filtering collections. They're case-sensitive, so combine with `lower()` when working with user-provided input that might have inconsistent casing.
