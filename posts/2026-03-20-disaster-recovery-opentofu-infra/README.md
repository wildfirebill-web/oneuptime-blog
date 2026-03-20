# How to Plan Disaster Recovery for OpenTofu-Managed Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Disaster Recovery, Infrastructure as Code, Business Continuity, DevOps, Resilience

Description: Learn how to leverage OpenTofu's code-driven approach to enable rapid infrastructure recovery, including state backup strategies, cross-region replication, and recovery runbooks.

## Introduction

OpenTofu's greatest disaster recovery advantage is that infrastructure is defined as code - if you lose your cloud environment, you can recreate it from configuration files. But DR planning requires more than just having the code: state files, secrets, and provider credentials must also be preserved and accessible in a failure scenario.

## What to Protect for DR

```text
1. Configuration files          → Git repository (replicated automatically)
2. State files                  → S3 with cross-region replication
3. State lock table             → DynamoDB (replicate or accept re-creation)
4. Secrets                      → AWS Secrets Manager with cross-region replication
5. Provider credentials         → IAM roles (tied to AWS, recreate in DR region)
6. Provider binaries            → Use plugin cache or private mirror
```

## Cross-Region State Replication

```hcl
# Replicate state bucket to a DR region

resource "aws_s3_bucket_replication_configuration" "state_dr" {
  bucket = aws_s3_bucket.state.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "state-dr-replication"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.state_dr.arn
      storage_class = "STANDARD_IA"  # Cost-optimized for DR

      # Use same KMS key in DR region (or a DR-region key)
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.state_dr.arn
      }
    }
  }
}

# DR region state bucket
resource "aws_s3_bucket" "state_dr" {
  provider = aws.dr_region
  bucket   = "my-opentofu-state-dr"
}
```

## Versioning for State Rollback

```hcl
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

Restore a previous state version:

```bash
# List state file versions
aws s3api list-object-versions \
  --bucket my-opentofu-state \
  --prefix prod/app/tofu.tfstate

# Download a specific version
aws s3api get-object \
  --bucket my-opentofu-state \
  --key prod/app/tofu.tfstate \
  --version-id "abc123" \
  tofu.tfstate.rollback
```

## DR Region Configuration

Maintain a parallel configuration for the DR region:

```text
environments/
├── prod/          → Primary region (us-east-1)
└── prod-dr/       → DR region (us-west-2) - same modules, different inputs
```

```hcl
# environments/prod-dr/terraform.tfvars
region    = "us-west-2"
az_count  = 2
# Scale down for standby: smaller instances, less capacity
instance_type = "t3.large"  # vs m5.2xlarge in primary
```

## Recovery Runbook

```bash
#!/bin/bash
# dr-recovery.sh - step-by-step recovery script

set -euo pipefail

DR_REGION="us-west-2"
STATE_BUCKET="my-opentofu-state-dr"

echo "Step 1: Verify DR state is accessible"
aws s3 ls "s3://${STATE_BUCKET}/prod-dr/" --region "$DR_REGION"

echo "Step 2: Initialize DR configuration"
cd environments/prod-dr
tofu init \
  -backend-config="bucket=${STATE_BUCKET}" \
  -backend-config="region=${DR_REGION}"

echo "Step 3: Plan DR environment"
tofu plan -var="region=${DR_REGION}"

echo "Step 4: Apply DR environment (requires human confirmation)"
# tofu apply -var="region=${DR_REGION}"
```

## RTO/RPO Targets

| Component | Recovery Target |
|---|---|
| State file | RPO: 1 hour (replication lag), RTO: < 5 min |
| Infrastructure | RTO: 30-90 minutes (tofu apply from scratch) |
| Data (RDS) | Depends on backup/replica strategy |

## Testing DR

```bash
# Regular DR drills - apply the DR config in the DR region
cd environments/prod-dr
tofu plan   # Verify the config is current and valid

# Full DR test (quarterly): apply everything in the DR region
tofu apply -var="environment=prod-dr"
# Verify the application works, then destroy DR resources
tofu destroy -var="environment=prod-dr"
```

## Conclusion

OpenTofu-managed infrastructure is inherently more recoverable than click-ops infrastructure - the configuration IS the recovery plan. Protect state files with cross-region replication and versioning, maintain a DR-region configuration that can be applied at any time, and run regular DR drills to verify that recovery actually works within your RTO targets.
