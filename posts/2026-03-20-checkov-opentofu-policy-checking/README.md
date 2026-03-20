# How to Use Checkov with OpenTofu for Policy Checking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Checkov, Policy, Compliance, Security

Description: Learn how to use Checkov to enforce organizational policies on OpenTofu configurations, write custom policies, and integrate policy checking into your deployment pipeline.

## Introduction

Checkov goes beyond basic security scanning to support custom organizational policies. Whether enforcing tagging standards, requiring specific encryption configurations, or validating compliance requirements, Checkov's Python and YAML-based custom checks let you encode your organization's rules as executable policies.

## Built-in Policy Scanning

```bash
# Install

pip install checkov

# Run with default policies
checkov -d . --framework terraform

# Run against specific compliance framework
checkov -d . --compliance PCI_DSS_V321

# List available compliance frameworks
checkov -d . --list-compliance

# Show only specific severity
checkov -d . --check MEDIUM,HIGH,CRITICAL
```

## Writing Custom Python Policies

```python
# policies/check_required_tags.py
from checkov.common.models.enums import CheckCategories, CheckResult
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

REQUIRED_TAGS = {"Environment", "Owner", "Project", "CostCenter"}

class CheckRequiredTags(BaseResourceCheck):
    def __init__(self):
        name = "Ensure required organizational tags are present"
        id = "CKV_MYORG_1"
        categories = [CheckCategories.GENERAL_SECURITY]
        # Apply to all taggable resource types
        supported_resources = [
            "aws_instance",
            "aws_s3_bucket",
            "aws_db_instance",
            "aws_eks_cluster",
            "aws_ecs_service",
            "aws_lambda_function",
        ]
        super().__init__(
            name=name, id=id,
            categories=categories,
            supported_resources=supported_resources
        )

    def scan_resource_conf(self, conf):
        tags = conf.get("tags", [{}])
        if isinstance(tags, list):
            tags = tags[0] if tags else {}

        if not isinstance(tags, dict):
            return CheckResult.FAILED

        missing_tags = REQUIRED_TAGS - set(tags.keys())
        if missing_tags:
            self.details.append(f"Missing required tags: {missing_tags}")
            return CheckResult.FAILED

        return CheckResult.PASSED

scanner = CheckRequiredTags()
```

## Writing Custom YAML Policies

```yaml
# policies/require_kms_encryption.yaml
metadata:
  name: "Require KMS Encryption on S3 Buckets"
  id: "CKV_MYORG_2"
  category: "ENCRYPTION"
  severity: "HIGH"

definition:
  cond_type: attribute
  resource_types:
    - aws_s3_bucket_server_side_encryption_configuration
  attribute: rule.apply_server_side_encryption_by_default.sse_algorithm
  operator: equals
  value: "aws:kms"
```

```yaml
# policies/enforce_vpc_flow_logs.yaml
metadata:
  name: "Ensure VPC Flow Logs are enabled"
  id: "CKV_MYORG_3"
  category: "LOGGING"
  severity: "MEDIUM"

definition:
  and:
    - cond_type: filter
      resource_types:
        - aws_flow_log
      filters:
        - attribute: vpc_id
          operator: exists
    - cond_type: attribute
      resource_types:
        - aws_flow_log
      attribute: traffic_type
      operator: equals
      value: "ALL"
```

## Running with Custom Policies

```bash
# Run with custom policy directory
checkov -d . --external-checks-dir=policies/ --framework terraform

# Run with specific custom check
checkov -d . --check=CKV_MYORG_1,CKV_MYORG_2

# Run all checks including custom
checkov -d . --external-checks-dir=policies/ --check=CKV_AWS_20,CKV_MYORG_1
```

## Checkov Configuration File

```yaml
# .checkov.yaml
framework:
  - terraform
directory:
  - .
check:
  - CKV_AWS_20    # S3 bucket ACL
  - CKV_AWS_18    # S3 access logging
  - CKV_AWS_57    # S3 versioning
  - CKV_AWS_145   # S3 KMS encryption
  - CKV_AWS_119   # DynamoDB KMS
  - CKV2_AWS_62   # S3 event notifications
  - CKV_MYORG_1   # Required tags
  - CKV_MYORG_2   # KMS encryption
  - CKV_MYORG_3   # VPC flow logs
external-checks-dir:
  - ./policies
soft-fail: false
output:
  - cli
  - sarif
output-file-path: checkov-results.sarif
```

## Inline Suppression

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "access-logs"

  # checkov:skip=CKV_AWS_57:Access logs bucket doesn't need versioning
  # checkov:skip=CKV_AWS_18:This IS the log bucket - no meta-logging required
}
```

## Baseline Mode (For Gradual Adoption)

```bash
# Generate a baseline of existing findings (accept current state)
checkov -d . -o json > .checkov.baseline

# Future runs only report NEW findings not in the baseline
checkov -d . --baseline .checkov.baseline
```

## GitHub Actions Integration

```yaml
      - name: Checkov policy check
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          external_checks_dir: policies/
          output_format: sarif
          output_file_path: reports/checkov.sarif
          soft_fail: false
          check: CKV_MYORG_1,CKV_MYORG_2,CKV_MYORG_3
```

## Conclusion

Checkov's custom policy support transforms it from a security scanner into a policy enforcement platform. Encode organizational requirements - required tags, approved encryption algorithms, logging mandates - as custom checks that run alongside the built-in rules. The baseline mode enables gradual adoption in existing codebases by accepting the current state and only blocking new violations, making it practical to add policy checking to established projects.
