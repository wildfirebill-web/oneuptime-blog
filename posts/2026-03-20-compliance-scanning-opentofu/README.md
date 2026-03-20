# How to Run Compliance Scanning on OpenTofu Configurations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Compliance, Security Scanning, CIS Benchmark, HIPAA, SOC2, Infrastructure as Code

Description: Learn how to scan OpenTofu configurations against compliance frameworks like CIS Benchmarks, HIPAA, and SOC 2 using Checkov and custom OPA policies — ensuring infrastructure meets regulatory requirements before deployment.

## Introduction

Compliance scanning automates the verification that infrastructure configurations meet regulatory and security framework requirements. Rather than relying on manual audits after deployment, scan OpenTofu configs against CIS Benchmarks, PCI DSS, HIPAA, and SOC 2 controls as part of every pull request.

## Checkov: Built-in Compliance Frameworks

Checkov includes built-in checks mapped to compliance frameworks:

```bash
# Install Checkov
pip install checkov

# Scan against CIS AWS Benchmark Level 1
checkov -d . --framework terraform --check CIS_AWS_1.2

# Scan against HIPAA controls
checkov -d . --framework terraform --bc-api-key $BRIDGECREW_TOKEN --compliance HIPAA

# Scan against SOC 2
checkov -d . --framework terraform --bc-api-key $BRIDGECREW_TOKEN --compliance SOC2

# Run all built-in checks and export results
checkov -d . --output json --output-file compliance-results.json
```

## Common CIS AWS Benchmark Violations

```hcl
# CKV_AWS_23: Ensure all data stored in the S3 bucket is securely encrypted at rest
# FIX: Enable default encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

# CKV_AWS_18: Ensure the S3 bucket has access logging enabled
resource "aws_s3_bucket_logging" "app" {
  bucket        = aws_s3_bucket.app.id
  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "app-access-logs/"
}

# CKV_AWS_86: Ensure RDS is not publicly accessible
resource "aws_db_instance" "postgres" {
  identifier         = "prod-postgres"
  publicly_accessible = false  # CIS requirement

  # CKV_AWS_157: Ensure RDS has deletion protection
  deletion_protection = true

  # CKV_AWS_133: Ensure RDS is encrypted
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
}
```

## Custom OPA Compliance Policies

Write Rego policies for controls that Checkov doesn't cover out of the box:

```rego
# policies/pci_dss.rego
package main

import future.keywords.in

# PCI DSS Req 1.3: Prohibit direct public access to cardholder data environment
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.actions[_] in ["create", "update"]

    # DB must not be in a public subnet
    resource.change.after.publicly_accessible == true

    msg := sprintf(
        "PCI DSS VIOLATION: RDS instance '%s' must not be publicly accessible",
        [resource.address]
    )
}

# PCI DSS Req 8.2: Encrypt all stored cardholder data
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.actions[_] in ["create", "update"]

    resource.change.after.storage_encrypted != true

    msg := sprintf(
        "PCI DSS VIOLATION: RDS instance '%s' must have storage_encrypted = true",
        [resource.address]
    )
}
```

## GitHub Actions: Compliance Gate

```yaml
name: Compliance Scan

on:
  pull_request:

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: OpenTofu Init & Plan
        run: |
          tofu init
          tofu plan -out=tfplan.binary
          tofu show -json tfplan.binary > tfplan.json
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Checkov CIS Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          check: CKV_AWS_23,CKV_AWS_18,CKV_AWS_86,CKV_AWS_157,CKV_AWS_133
          output_format: sarif
          output_file_path: checkov-results.sarif

      - name: OPA Custom Compliance Policies
        run: |
          brew install conftest
          conftest test tfplan.json --policy policies/

      - name: Upload Checkov Results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov-results.sarif
```

## Compliance as Code: .checkov.yaml

```yaml
# .checkov.yaml — project-level Checkov configuration
framework:
  - terraform

check:
  - CIS_AWS_1.2   # CIS Benchmark Level 1
  - PCI_DSS_3.2   # PCI DSS

skip-check:
  # Public website bucket — intentionally public
  - CKV_AWS_20
  - CKV2_AWS_6

compact: true
output: cli,sarif
output-file-path: .
```

## Generating Compliance Reports

```bash
# Generate a JSON compliance report for auditors
checkov -d . \
  --output json \
  --output-file compliance-report.json \
  --soft-fail   # Don't exit non-zero (just report)

# Extract passing/failing check counts
jq '.summary | {passed: .passed, failed: .failed}' compliance-report.json
# {"passed": 142, "failed": 3}
```

## Conclusion

Compliance scanning in OpenTofu shifts regulatory verification left — from post-deployment audits to pre-merge checks. Checkov's 1,000+ built-in controls cover CIS Benchmarks, PCI DSS, HIPAA, and SOC 2. Supplement with custom OPA policies for organization-specific controls. Generate SARIF output for GitHub Security tab integration and JSON reports for auditors. With compliance gates in CI, every merged change is compliance-verified.
