# How to Use the fileset Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the fileset function in OpenTofu to list files matching a glob pattern for bulk S3 uploads, config deployment, and multi-file operations.

## Introduction

The `fileset` function in OpenTofu returns a set of file paths matching a glob pattern within a given directory. It is the foundation of data-driven multi-file operations — uploading all files in a directory to S3, deploying all Kubernetes manifests, or reading all config files.

## Syntax

```hcl
fileset(path, pattern)
```

- **path** — the base directory to search in
- **pattern** — a glob pattern relative to `path`
- Returns a `set(string)` of relative paths

## Basic Examples

```hcl
output "all_tf_files" {
  value = fileset(path.module, "*.tf")
}

output "all_json_configs" {
  value = fileset("${path.module}/config", "**/*.json")
}

output "html_files" {
  value = fileset("${path.module}/web", "**/*.html")
}
```

## Practical Use Cases

### Bulk S3 Upload (Static Website)

```hcl
locals {
  static_files = fileset("${path.module}/web/dist", "**")
}

resource "aws_s3_object" "static" {
  for_each = local.static_files

  bucket = aws_s3_bucket.website.id
  key    = each.key
  source = "${path.module}/web/dist/${each.key}"

  content_type = lookup({
    "html" = "text/html"
    "css"  = "text/css"
    "js"   = "application/javascript"
    "png"  = "image/png"
    "jpg"  = "image/jpeg"
  }, reverse(split(".", each.key))[0], "application/octet-stream")

  etag = filemd5("${path.module}/web/dist/${each.key}")
}
```

### Deploying All Kubernetes Manifests

```hcl
locals {
  manifest_files = fileset("${path.module}/k8s", "*.yaml")
}

resource "kubectl_manifest" "app" {
  for_each  = local.manifest_files
  yaml_body = file("${path.module}/k8s/${each.key}")
}
```

### Loading All Config Files

```hcl
locals {
  config_files = fileset("${path.module}/configs", "*.json")

  # Load and merge all config files
  all_configs = merge([
    for f in local.config_files :
    jsondecode(file("${path.module}/configs/${f}"))
  ]...)
}
```

### Lambda Layer Contents

```hcl
locals {
  layer_files = fileset("${path.module}/layer", "**")
}

resource "aws_s3_object" "layer_files" {
  for_each = local.layer_files

  bucket = aws_s3_bucket.lambda_layers.id
  key    = "layers/my-layer/${each.key}"
  source = "${path.module}/layer/${each.key}"
}
```

### Change Detection for All Source Files

```hcl
locals {
  source_files = fileset("${path.module}/src", "**/*.py")

  source_hash = sha256(join("", [
    for f in sort(tolist(local.source_files)) :
    filesha256("${path.module}/src/${f}")
  ]))
}

resource "null_resource" "rebuild" {
  triggers = {
    source_hash = local.source_hash
  }

  provisioner "local-exec" {
    command = "pip install -r requirements.txt && zip -r function.zip src/"
    working_dir = path.module
  }
}
```

## Step-by-Step Usage

1. Identify the directory and file pattern you need.
2. Call `fileset(path, pattern)`.
3. Use `for_each = fileset(...)` for direct iteration, or convert to list with `tolist()`.
4. Test in `tofu console`:

```bash
tofu console

> fileset(".", "*.tf")
toset(["main.tf", "variables.tf", "outputs.tf"])
```

## Glob Patterns

| Pattern | Matches |
|---------|---------|
| `*.json` | JSON files in base dir only |
| `**/*.json` | JSON files in all subdirectories |
| `configs/**` | All files under configs/ |
| `*.{yaml,yml}` | Both YAML extensions |

## Conclusion

The `fileset` function is OpenTofu's directory scanner, enabling data-driven resource creation for bulk file operations. It is essential for static website deployments, Kubernetes manifest application, and change detection across file collections. Combined with `for_each`, it creates one resource per file with minimal code.
