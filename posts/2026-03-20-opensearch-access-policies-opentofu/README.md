# How to Configure OpenSearch Access Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, OpenSearch, Access Control, Infrastructure as Code

Description: Learn how to configure AWS OpenSearch Service access policies with OpenTofu for fine-grained access control using IAM and resource-based policies.

OpenSearch access policies control who can call OpenSearch API actions on your domain. Combining resource-based domain policies with fine-grained access control (FGAC) lets you manage access at the index and document level. Managing these in OpenTofu keeps security configuration version-controlled.

## Domain Access Policy

```hcl
resource "aws_opensearch_domain_policy" "main" {
  domain_name = aws_opensearch_domain.main.domain_name

  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # Allow app role to read/write
      {
        Sid    = "AllowAppAccess"
        Effect = "Allow"
        Principal = {
          AWS = [
            aws_iam_role.app.arn,
            aws_iam_role.log_shipper.arn,
          ]
        }
        Action = [
          "es:ESHttpGet",
          "es:ESHttpPost",
          "es:ESHttpPut",
          "es:ESHttpDelete",
          "es:ESHttpHead",
          "es:ESHttpPatch",
        ]
        Resource = "${aws_opensearch_domain.main.arn}/*"
      },
      # Allow admin role full access
      {
        Sid    = "AllowAdminAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.opensearch_admin.arn
        }
        Action   = "es:*"
        Resource = "${aws_opensearch_domain.main.arn}/*"
      },
      # Deny access from specific IPs (e.g., block known bad actors)
      {
        Sid    = "DenyBadActors"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action   = "es:*"
        Resource = "${aws_opensearch_domain.main.arn}/*"
        Condition = {
          IpAddress = {
            "aws:SourceIp" = ["198.51.100.0/24"]
          }
        }
      }
    ]
  })
}
```

## Fine-Grained Access Control (FGAC)

With FGAC enabled, you configure permissions inside OpenSearch using roles and role mappings. The OpenTofu resource enables FGAC with a master user:

```hcl
resource "aws_opensearch_domain" "secure" {
  domain_name    = "secure-search"
  engine_version = "OpenSearch_2.11"

  cluster_config {
    instance_type  = "r6g.large.search"
    instance_count = 3
  }

  ebs_options {
    ebs_enabled = true
    volume_size = 50
  }

  encrypt_at_rest     { enabled = true }
  node_to_node_encryption { enabled = true }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  # Enable Fine-Grained Access Control
  advanced_security_options {
    enabled                        = true
    anonymous_auth_enabled         = false
    internal_user_database_enabled = true  # Use internal user DB

    master_user_options {
      master_user_name     = "admin"
      master_user_password = var.opensearch_admin_password
    }
  }
}
```

## IAM Role for OpenSearch Access

```hcl
resource "aws_iam_role" "search_reader" {
  name = "opensearch-reader"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "search_reader" {
  role = aws_iam_role.search_reader.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["es:ESHttpGet", "es:ESHttpPost"]
      Resource = "${aws_opensearch_domain.main.arn}/*"
    }]
  })
}
```

## VPC Endpoint Policy

```hcl
# Restrict access to specific VPC only
resource "aws_opensearch_domain_policy" "vpc_only" {
  domain_name = aws_opensearch_domain.main.domain_name

  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { AWS = "*" }
        Action    = "es:*"
        Resource  = "${aws_opensearch_domain.main.arn}/*"
        Condition = {
          StringEquals = {
            "aws:SourceVpc" = var.vpc_id
          }
        }
      }
    ]
  })
}
```

## Conclusion

OpenSearch access policies in OpenTofu give you version-controlled, auditable access control. Define resource-based domain policies for IAM-level access, enable fine-grained access control for index and document-level permissions, and restrict access to your VPC using vpc_options with a VPC-scoped condition in the access policy. Grant least-privilege permissions — read-only for consumers, write for shippers, admin only for operations.
