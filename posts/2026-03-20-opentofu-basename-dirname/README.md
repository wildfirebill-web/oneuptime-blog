# How to Use the basename and dirname Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the basename and dirname functions in OpenTofu to extract file names and directory paths from full file paths.

## Introduction

The `basename` and `dirname` functions in OpenTofu extract components from filesystem paths. `basename` returns the final element of a path (the filename), while `dirname` returns the directory portion (everything except the final element).

## Syntax

```hcl
basename(path)
dirname(path)
```

## Basic Examples

```hcl
output "basename_example" {
  value = basename("/path/to/file.txt")  # Returns "file.txt"
}

output "dirname_example" {
  value = dirname("/path/to/file.txt")   # Returns "/path/to"
}

output "basename_no_extension" {
  value = basename("/etc/nginx/nginx.conf")  # Returns "nginx.conf"
}

output "dirname_root" {
  value = dirname("/etc/nginx")  # Returns "/etc"
}
```

## Practical Use Cases

### Extracting Config File Name from Path

```hcl
variable "config_path" {
  type    = string
  default = "/app/configs/production.json"
}

locals {
  config_filename = basename(var.config_path)  # "production.json"
  config_dir      = dirname(var.config_path)   # "/app/configs"
}

resource "aws_ssm_parameter" "config_name" {
  name  = "/app/config-filename"
  type  = "String"
  value = local.config_filename
}
```

### Processing fileset Results

```hcl
locals {
  config_files = fileset("${path.module}/configs", "**/*.json")
}

resource "aws_s3_object" "configs" {
  for_each = local.config_files

  bucket = aws_s3_bucket.config.id
  # Use basename as the S3 key
  key    = basename(each.key)
  source = "${path.module}/configs/${each.key}"
}
```

### Dynamic S3 Key Generation

```hcl
variable "source_files" {
  type    = list(string)
  default = [
    "/local/path/app.jar",
    "/local/path/config.yaml",
    "/local/path/data.csv"
  ]
}

locals {
  s3_uploads = {
    for path in var.source_files :
    basename(path) => path
  }
}

resource "aws_s3_object" "artifacts" {
  for_each = local.s3_uploads

  bucket = aws_s3_bucket.artifacts.id
  key    = "releases/${each.key}"
  source = each.value
}
```

### Grouping Files by Directory

```hcl
variable "file_paths" {
  type = list(string)
  default = [
    "configs/prod/app.yaml",
    "configs/prod/db.yaml",
    "configs/dev/app.yaml"
  ]
}

locals {
  files_by_dir = {
    for p in var.file_paths :
    dirname(p) => basename(p)...
  }
}
```

## Step-by-Step Usage

```bash
tofu console

> basename("/usr/local/bin/tofu")
"tofu"
> dirname("/usr/local/bin/tofu")
"/usr/local/bin"
> basename("config.tf")
"config.tf"
> dirname("config.tf")
"."
```

## Removing File Extensions

OpenTofu doesn't have a built-in extension-stripping function, but you can combine with `trimsuffix`:

```hcl
locals {
  filename_without_ext = trimsuffix(basename("/path/to/file.json"), ".json")
  # Returns "file"
}
```

## Conclusion

The `basename` and `dirname` functions are essential for path manipulation in OpenTofu. Use `basename` when you need just the filename from a full path, and `dirname` when you need the containing directory. These are particularly useful when processing `fileset()` output or dynamically generating S3 object keys from local file paths.
