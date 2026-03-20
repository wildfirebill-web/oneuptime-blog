# How to Use the trimspace Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the trimspace function in OpenTofu to remove leading and trailing whitespace from strings for clean resource names and values.

## Introduction

The `trimspace` function in OpenTofu removes all leading and trailing whitespace characters from a string, including spaces, tabs, and newlines. It is the most common string cleaning function and is essential for processing user inputs, file contents, and external data.

## Syntax

```hcl
trimspace(string)
```

- Removes all leading and trailing whitespace
- Whitespace includes spaces (` `), tabs (`\t`), newlines (`\n`), carriage returns (`\r`)
- Internal whitespace is preserved

## Basic Examples

```hcl
output "trim_spaces" {
  value = trimspace("  hello  ")     # Returns "hello"
}

output "trim_tabs" {
  value = trimspace("\thello\t")     # Returns "hello"
}

output "trim_newlines" {
  value = trimspace("\nhello\n")     # Returns "hello"
}

output "preserve_internal" {
  value = trimspace("  hello world  ")  # Returns "hello world"
}

output "already_clean" {
  value = trimspace("hello")         # Returns "hello" (unchanged)
}
```

## Practical Use Cases

### Cleaning Variable Inputs

```hcl
variable "bucket_name" {
  type        = string
  description = "S3 bucket name (spaces will be trimmed)"
  default     = "  my-app-bucket  "
}

locals {
  clean_bucket_name = trimspace(var.bucket_name)
}

resource "aws_s3_bucket" "app" {
  bucket = local.clean_bucket_name  # "my-app-bucket"
}
```

### Reading Configuration Files

```hcl
# config.txt contains: "  us-east-1  \n"
locals {
  region = trimspace(file("${path.module}/config/region.txt"))
}

provider "aws" {
  region = local.region  # "us-east-1"
}
```

### Processing Tag Values

```hcl
variable "tags" {
  type = map(string)
  default = {
    team        = "  Platform Engineering  "
    owner       = "\tDevOps Team\t"
    cost_center = "  CC-1234  "
  }
}

locals {
  # Clean all tag values
  clean_tags = {
    for k, v in var.tags :
    k => trimspace(v)
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  tags          = local.clean_tags
}
```

### Environment Variable Processing

```hcl
data "external" "env_vars" {
  program = ["bash", "-c", <<-EOF
    echo '{"region": "us-east-1 ", "env": " production"}'
  EOF
  ]
}

locals {
  region      = trimspace(data.external.env_vars.result["region"])
  environment = trimspace(data.external.env_vars.result["env"])
}
```

### Multi-line String Cleanup

```hcl
locals {
  raw_script = <<-EOF
    
    #!/bin/bash
    echo "Hello World"
    
  EOF

  # trimspace removes leading/trailing blank lines and whitespace
  clean_script = trimspace(local.raw_script)
}

output "clean_script" {
  value = local.clean_script
  # Returns: "#!/bin/bash\necho \"Hello World\""
}
```

## Step-by-Step Usage

1. Identify strings that may have leading/trailing whitespace.
2. Apply `trimspace()` to clean them.
3. Use the cleaned value in resource definitions.
4. Test in `tofu console`:

```bash
tofu console

> trimspace("  hello world  ")
"hello world"
> trimspace("\n\thello\n\t")
"hello"
```

## Combining trimspace with Other Functions

```hcl
locals {
  raw_input = "  My Service Name  "
  # Clean, then convert to lowercase slug
  slug = replace(lower(trimspace(local.raw_input)), " ", "-")
  # Returns: "my-service-name"
}
```

## trimspace vs chomp vs trim

| Function | Removes |
|----------|---------|
| `trimspace(s)` | All leading/trailing whitespace (spaces, tabs, newlines) |
| `chomp(s)` | Only trailing newlines |
| `trim(s, chars)` | Specified characters from both ends |

## Conclusion

The `trimspace` function is the most universally applicable string cleaning function in OpenTofu. Make it a defensive habit to apply `trimspace` to all string inputs from variables, files, and external sources before using them in resource definitions. This prevents subtle whitespace-related issues that can be difficult to debug in infrastructure configurations.
