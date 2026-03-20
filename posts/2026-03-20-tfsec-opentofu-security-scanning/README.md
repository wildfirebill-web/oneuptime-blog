# How to Use tfsec with OpenTofu for Security Scanning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, tfsec, Security Scanning, SAST, Compliance

Description: Learn how to use tfsec to scan OpenTofu configurations for security misconfigurations, with custom checks and CI/CD integration for continuous security validation.

## Introduction

tfsec is a static security analysis tool for Terraform and OpenTofu configurations. It scans for common security misconfigurations across AWS, Azure, GCP, and Kubernetes resources, providing fast feedback without cloud credentials. Note: tfsec has been absorbed into Trivy but remains available as a standalone tool.

## Installation

```bash
# macOS
brew install tfsec

# Linux
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Or via Go
go install github.com/aquasecurity/tfsec/cmd/tfsec@latest

# Verify
tfsec --version
```

## Basic Usage

```bash
# Scan current directory
tfsec .

# Scan with specific format
tfsec . --format=json > tfsec-results.json

# Scan with severity threshold (only show HIGH and CRITICAL)
tfsec . --minimum-severity HIGH

# Scan and exit 0 even with findings (for gradual adoption)
tfsec . --soft-fail

# Output as SARIF for GitHub Security tab
tfsec . --format=sarif --out tfsec.sarif
```

## Common Security Issues tfsec Catches

```hcl
# tfsec: aws-s3-no-public-access-with-acl
# FAIL: S3 bucket with public ACL
resource "aws_s3_bucket_acl" "example" {
  bucket = aws_s3_bucket.main.id
  acl    = "public-read"  # Security risk
}

# FIX: Keep private
resource "aws_s3_bucket_acl" "example" {
  bucket = aws_s3_bucket.main.id
  acl    = "private"
}

# tfsec: aws-ec2-no-public-ip
# FAIL: Subnet auto-assigns public IPs
resource "aws_subnet" "public" {
  map_public_ip_on_launch = true  # Acceptable for public subnets, but flagged
}

# tfsec: aws-rds-enable-deletion-protection
# FAIL: RDS without deletion protection
resource "aws_db_instance" "main" {
  # deletion_protection not set (defaults to false)
  identifier = "my-db"
}

# FIX:
resource "aws_db_instance" "main" {
  deletion_protection = true
  identifier          = "my-db"
}

# tfsec: aws-iam-no-policy-wildcards
# FAIL: Overly permissive IAM policy
resource "aws_iam_policy" "admin" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["*"]       # Wildcard action
      Resource = ["*"]       # Wildcard resource
    }]
  })
}
```

## Inline Suppression

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "access-logs-bucket"

  # tfsec:ignore:aws-s3-enable-versioning
  # Justification: Access logs don't need versioning
}

# Or with expiry date (self-expiring suppression)
# tfsec:ignore:aws-ec2-no-public-ip:exp:2026-06-01
resource "aws_subnet" "bastion" {
  map_public_ip_on_launch = true
}
```

## Configuration File

```yaml
# .tfsec/config.yml
exclude:
  - aws-s3-enable-versioning          # Not required for log buckets
  - aws-s3-enable-bucket-logging       # Logging buckets don't log themselves

minimum_severity: MEDIUM

custom_checks:
  - action: DENY
    description: "Enforce KMS encryption on all S3 buckets"
    error_message: "S3 bucket must use KMS encryption"
    impact: "Data at rest may not be encrypted with customer-managed keys"
    resolution: "Add server_side_encryption_configuration with KMS"
    required_types:
      - resource
    required_labels:
      - aws_s3_bucket
    checks:
      - attribute: "server_side_encryption_configuration.rule.apply_server_side_encryption_by_default.sse_algorithm"
        equals: "aws:kms"
```

## Custom Checks in Rego

```rego
# .tfsec/custom_checks/require_kms.rego
package custom.aws.s3

import future.keywords.contains

deny contains msg if {
    bucket := input.config.resource.aws_s3_bucket[name]
    not bucket.server_side_encryption_configuration
    msg := sprintf("S3 bucket %s must have KMS encryption enabled", [name])
}
```

## CI/CD Integration

```yaml
# .github/workflows/tfsec.yml
name: tfsec Security Scan

on: [pull_request]

jobs:
  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: .
          minimum_severity: MEDIUM
          soft_fail: false
          format: sarif
          output: tfsec.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: tfsec.sarif
```

## tfsec vs Checkov vs Trivy

For new projects, Trivy is recommended as it absorbs tfsec's functionality while adding container scanning. For existing projects using tfsec, migration to Trivy is straightforward since Trivy uses the same check IDs.

```bash
# Trivy with the same tfsec-style output
trivy config --format table .
```

## Conclusion

tfsec provides fast security scanning feedback on OpenTofu configurations. The inline suppression with `# tfsec:ignore:rule-id` is useful for legitimate exceptions, but always add a justification comment explaining why the finding is acceptable. For new projects, consider using Trivy directly as it includes tfsec's checks plus container scanning in a single tool.
