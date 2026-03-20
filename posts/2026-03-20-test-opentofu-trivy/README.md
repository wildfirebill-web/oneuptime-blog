# How to Test OpenTofu Configurations with Trivy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Trivy, Security Scanning, Vulnerability, Compliance

Description: Learn how to use Trivy to scan OpenTofu configurations for security misconfigurations and compliance issues, with support for CIS benchmarks and custom policies.

## Introduction

Trivy is a comprehensive security scanner from Aqua Security that scans container images, filesystems, and infrastructure-as-code including OpenTofu/Terraform configurations. It checks for security misconfigurations against built-in policies and supports custom Rego policies for organization-specific rules.

## Installation and Basic Scanning

```bash
# Install Trivy

brew install trivy  # macOS
# or
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan current directory
trivy config .

# Scan with severity filter
trivy config --severity HIGH,CRITICAL .

# Scan a specific file
trivy config main.tf

# Output as JSON
trivy config -f json -o trivy-results.json .

# Output as SARIF (for GitHub Security tab)
trivy config -f sarif -o trivy-results.sarif .
```

## Understanding Output

```bash
# Example output:
# main.tf (terraform)
# ==================
# Tests: 20 (SUCCESSES: 18, FAILURES: 2, EXCEPTIONS: 0)
# Failures: 2 (HIGH: 1, CRITICAL: 1)
#
# CRITICAL: Security group should restrict access on port 22
# ════════════════════════════════════════════
#  ID             CVE-2022-0001
#  File           main.tf:15-25
#  Message        Security group allows SSH access from the internet
```

## Common OpenTofu Issues Trivy Catches

```hcl
# Trivy check: AVD-AWS-0057 - S3 bucket without versioning
# FIX:
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Trivy check: AVD-AWS-0086 - Security group allows unrestricted SSH
# FIX: Restrict SSH to specific CIDRs
resource "aws_security_group_rule" "ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["10.0.0.0/8"]  # Internal only
  # NOT: cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.main.id
}

# Trivy check: AVD-AWS-0132 - RDS without deletion protection
# FIX:
resource "aws_db_instance" "main" {
  # ...
  deletion_protection = true
}
```

## Custom Rego Policies

```rego
# policies/require_tags.rego
package user.terraform.require_tags

import future.keywords.contains
import future.keywords.if

deny contains msg if {
    resource := input.config.resource.aws_instance[_]
    not resource.tags.Environment
    msg := sprintf("aws_instance must have an Environment tag: %s", [resource.name])
}

deny contains msg if {
    resource := input.config.resource.aws_s3_bucket[_]
    not resource.tags.Owner
    msg := sprintf("aws_s3_bucket must have an Owner tag: %s", [resource.name])
}
```

```bash
# Use custom policy directory
trivy config --policy policies/ .
```

## Trivy Configuration File

```yaml
# .trivyignore.yaml - ignore specific checks
rules:
  - id: AVD-AWS-0144  # S3 cross-region replication - not required
    reason: "Cross-region replication not required for dev environment"
    paths:
      - "environments/dev/**"
```

```bash
# Or use inline suppression comment in .tf files
# trivy:ignore:AVD-AWS-0057
resource "aws_s3_bucket" "logs" {
  bucket = "log-bucket"
  # Versioning not needed for access logs
}
```

## CIS Benchmark Scanning

```bash
# Run against CIS AWS Foundations Benchmark
trivy config --compliance aws-cis-1.2 .

# Run against SOC2 compliance
trivy config --compliance soc2 .

# List available compliance frameworks
trivy config --list-all-policies
```

## CI/CD Integration

```yaml
# .github/workflows/trivy-scan.yml
name: Trivy Security Scan

on: [pull_request]

jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy config scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: '1'

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

## Comparing Trivy vs Checkov

| Feature | Trivy | Checkov |
|---------|-------|---------|
| IaC scanning | Yes | Yes |
| Container scanning | Yes | No |
| Custom policies | Rego | Python/YAML |
| CIS Benchmarks | Yes | Yes |
| Speed | Fast | Fast |
| Output formats | JSON, SARIF, table | JSON, SARIF, JUnit |

## Conclusion

Trivy is particularly valuable when you want a single security scanner across both container images and infrastructure code - the same tool that scans your Docker images can scan your OpenTofu configurations. Use the SARIF output for GitHub Security tab integration, and apply `.trivyignore.yaml` for false positives with documented reasons rather than silently ignoring findings.
