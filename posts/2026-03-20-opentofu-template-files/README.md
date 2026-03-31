# How to Use Template Files for Dynamic Configuration in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, templatefile, Template, Dynamic Configuration, User Data

Description: Learn how to use templatefile() in OpenTofu to generate dynamic configuration files, user data scripts, and policy documents from templates with variable substitution.

## Overview

`templatefile(path, vars)` reads a file and replaces template variables (`${var}`) with provided values. This separates configuration templates from OpenTofu code, making it easier to manage complex multi-line configurations.

## Step 1: Basic Template Usage

```hcl
# main.tf - templatefile for dynamic scripts

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "m5.large"

  user_data = templatefile("${path.module}/templates/user-data.sh", {
    environment  = var.environment
    region       = var.region
    db_host      = aws_db_instance.app.address
    db_name      = "appdb"
    app_version  = var.app_version
    s3_bucket    = aws_s3_bucket.app.bucket
  })
}
```

```bash
# templates/user-data.sh
#!/bin/bash
set -e

# Configure application
cat > /etc/app/config.env << 'ENVFILE'
ENVIRONMENT=${environment}
AWS_REGION=${region}
DB_HOST=${db_host}
DB_NAME=${db_name}
APP_VERSION=${app_version}
S3_BUCKET=${s3_bucket}
ENVFILE

# Download and install application
aws s3 cp s3://${s3_bucket}/releases/${app_version}/app.tar.gz /tmp/
tar -xzf /tmp/app.tar.gz -C /opt/app/
systemctl restart app
```

## Step 2: Templates with Loops

```hcl
# Generate nginx config with multiple server blocks
resource "aws_instance" "nginx" {
  user_data = templatefile("${path.module}/templates/nginx.conf.tpl", {
    upstream_servers = [
      { name = "app1", host = "10.0.1.10", port = 8080 },
      { name = "app2", host = "10.0.1.11", port = 8080 },
      { name = "app3", host = "10.0.1.12", port = 8080 },
    ]
    worker_processes = 4
    server_name      = "app.example.com"
  })
}
```

```nginx
# templates/nginx.conf.tpl
worker_processes ${worker_processes};

http {
    upstream backend {
%{ for server in upstream_servers ~}
        server ${server.host}:${server.port};
%{ endfor ~}
    }

    server {
        listen 80;
        server_name ${server_name};
        location / {
            proxy_pass http://backend;
        }
    }
}
```

## Step 3: IAM Policy Templates

```hcl
# S3 bucket policy from template
resource "aws_s3_bucket_policy" "app" {
  bucket = aws_s3_bucket.app.id
  policy = templatefile("${path.module}/templates/s3-policy.json.tpl", {
    bucket_arn   = aws_s3_bucket.app.arn
    account_id   = data.aws_caller_identity.current.account_id
    allowed_vpce = aws_vpc_endpoint.s3.id
  })
}
```

```json
// templates/s3-policy.json.tpl
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "${bucket_arn}",
        "${bucket_arn}/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "${allowed_vpce}"
        }
      }
    }
  ]
}
```

## Step 4: Kubernetes ConfigMap from Template

```hcl
# Application config from template
resource "kubernetes_config_map" "app" {
  metadata {
    name      = "app-config"
    namespace = "production"
  }

  data = {
    "config.yaml" = templatefile("${path.module}/templates/app-config.yaml.tpl", {
      environment = var.environment
      db_host     = aws_db_instance.app.address
      redis_host  = aws_elasticache_replication_group.app.primary_endpoint_address
      log_level   = var.environment == "production" ? "WARN" : "DEBUG"
      features    = var.feature_flags
    })
  }
}
```

## Summary

`templatefile()` in OpenTofu separates complex configuration templates from infrastructure code. Template syntax supports conditional expressions (`%{ if condition }...%{ endif }`) and iteration (`%{ for item in list }...%{ endfor }`), making it possible to generate dynamic nginx configs, cloud-init scripts, and policy documents. The `~` strip marker removes whitespace after directives, keeping generated output clean. Templates should be version-controlled alongside the OpenTofu code they're used with.
