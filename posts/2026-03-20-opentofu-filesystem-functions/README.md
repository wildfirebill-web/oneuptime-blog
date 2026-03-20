---
title: "How to Use Filesystem Functions in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, functions, filesystem
description: "Learn how to use abspath, basename, dirname, file, filebase64, fileexists, fileset, and pathexpand in OpenTofu."
---

# How to Use Filesystem Functions in OpenTofu

OpenTofu provides filesystem functions that let you read files, inspect paths, and work with file contents during configuration evaluation. These are evaluated at plan/apply time on the machine running OpenTofu.

## abspath()

Returns the absolute path of a given path:

```hcl
> abspath("./scripts/init.sh")
"/home/user/projects/myapp/scripts/init.sh"
```

## basename()

Returns the last component of a path (filename):

```hcl
> basename("/home/user/projects/myapp/main.tf")
"main.tf"

> basename("path/to/module/")
"module"
```

## dirname()

Returns the directory containing a given path:

```hcl
> dirname("/home/user/projects/myapp/main.tf")
"/home/user/projects/myapp"

> dirname("path/to/file.txt")
"path/to"
```

```hcl
locals {
  # Reference files relative to the current module
  scripts_dir = "${dirname(abspath(path.module))}/shared-scripts"
}
```

## file()

Reads the content of a file as a string:

```hcl
# Read SSH public key for EC2 key pair
resource "aws_key_pair" "deploy" {
  key_name   = "deploy-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

# Read a startup script
resource "aws_instance" "web" {
  user_data = file("${path.module}/scripts/userdata.sh")
}

# Read a policy document
resource "aws_iam_role_policy" "app" {
  name   = "app-policy"
  role   = aws_iam_role.app.id
  policy = file("${path.module}/policies/app-policy.json")
}
```

## filebase64()

Reads file content and returns it Base64-encoded:

```hcl
# Encode a file for use in cloud-init or user data
resource "aws_instance" "app" {
  user_data_base64 = filebase64("${path.module}/scripts/init.sh")
}

# Provide a zip file as Base64 for Lambda
resource "aws_lambda_function" "app" {
  filename         = "${path.module}/function.zip"
  source_code_hash = filebase64sha256("${path.module}/function.zip")
  function_name    = "my-function"
  runtime          = "python3.11"
  handler          = "main.handler"
  role             = aws_iam_role.lambda.arn
}
```

## fileexists()

Returns true if a file exists at the given path:

```hcl
locals {
  # Use override file if it exists, otherwise use default
  config_file = fileexists("${path.module}/config.override.json") ? 
    "${path.module}/config.override.json" : 
    "${path.module}/config.default.json"
  
  config = jsondecode(file(local.config_file))
}
```

## fileset()

Returns a set of file paths matching a glob pattern:

```hcl
# Upload all HTML files in a directory
locals {
  html_files = fileset("${path.module}/website", "**/*.html")
}

resource "aws_s3_object" "website" {
  for_each = local.html_files

  bucket       = aws_s3_bucket.website.id
  key          = each.value
  source       = "${path.module}/website/${each.value}"
  content_type = "text/html"
  etag         = filemd5("${path.module}/website/${each.value}")
}
```

## pathexpand()

Expands `~` to the user's home directory:

```hcl
> pathexpand("~/.ssh/id_rsa.pub")
"/home/user/.ssh/id_rsa.pub"

resource "aws_key_pair" "deploy" {
  key_name   = "deploy"
  public_key = file(pathexpand("~/.ssh/id_rsa.pub"))
}
```

## Conclusion

Filesystem functions bridge your local files and OpenTofu configurations. Use `file()` for reading configuration and scripts, `filebase64()` for binary files and user data, `fileset()` for bulk file operations like S3 website uploads, and `fileexists()` for optional configuration files. Always use `path.module` or `path.root` for relative paths to ensure they resolve correctly regardless of where you run OpenTofu.
