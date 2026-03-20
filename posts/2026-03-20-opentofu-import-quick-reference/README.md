# How to Use the OpenTofu Import Quick Reference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Import, State Management, Quick Reference, Infrastructure as Code

Description: A quick reference for importing existing cloud resources into OpenTofu state using import blocks and the generate-config-out flag.

## Introduction

OpenTofu's import feature brings existing cloud resources under management without recreating them. This reference covers all import methods, common resource ID formats, and the auto-generate configuration workflow.

## Import Block (Recommended)

```hcl
# imports.tf

# Basic import

import {
  id = "my-existing-bucket"
  to = aws_s3_bucket.main
}

# Import with for_each (OpenTofu 1.7+)
locals {
  existing_buckets = {
    logs    = "company-logs-bucket"
    backups = "company-backups-bucket"
  }
}

import {
  for_each = local.existing_buckets
  id       = each.value
  to       = aws_s3_bucket.existing[each.key]
}
```

## CLI Import (Legacy)

```bash
# Single resource import via CLI
tofu import aws_s3_bucket.main my-existing-bucket

# Import with a specific state file
tofu import -state=terraform.tfstate aws_s3_bucket.main my-existing-bucket

# Import a for_each resource instance
tofu import 'aws_s3_bucket.existing["logs"]' company-logs-bucket
```

## Auto-Generate Configuration

```bash
# Define import blocks, then generate HCL automatically
tofu plan -generate-config-out=generated.tf

# Review and edit the generated file
# Then apply
tofu apply

# Remove import blocks after successful import
```

## Common AWS Import ID Formats

```javascript
Resource Type                    Import ID
---------------------------------   --------------------------
aws_s3_bucket                    bucket-name
aws_vpc                          vpc-0abc12345def67890
aws_subnet                       subnet-0abc12345def67890
aws_security_group               sg-0abc12345def67890
aws_internet_gateway             igw-0abc12345def67890
aws_route_table                  rtb-0abc12345def67890
aws_instance                     i-0abc12345def67890
aws_db_instance                  db-instance-identifier
aws_rds_cluster                  cluster-identifier
aws_iam_role                     role-name
aws_iam_policy                   arn:aws:iam::123456789012:policy/PolicyName
aws_iam_user                     username
aws_lambda_function              function-name
aws_ecs_cluster                  cluster-name
aws_ecs_service                  cluster-name/service-name
aws_eks_cluster                  cluster-name
aws_cloudfront_distribution      distribution-id (E1234ABCD...)
aws_route53_record               zone-id_record-name_record-type
aws_acm_certificate              arn:aws:acm:region:account:certificate/uuid
aws_kms_key                      key-id
aws_secretsmanager_secret        secret-arn
aws_sns_topic                    topic-arn
aws_sqs_queue                    queue-url
```

## Common Azure Import ID Formats

```text
Resource Type                    Import ID
---------------------------------   ---------------------------
azurerm_resource_group           /subscriptions/{sub}/resourceGroups/{name}
azurerm_storage_account          /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{name}
azurerm_virtual_network          /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{name}
azurerm_subnet                   /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{name}
```

## Common GCP Import ID Formats

```text
Resource Type                    Import ID
---------------------------------   ---------------------------
google_storage_bucket            project/bucket-name
google_compute_instance          projects/{project}/zones/{zone}/instances/{name}
google_compute_network           projects/{project}/global/networks/{name}
google_sql_database_instance     project/instance-name
```

## Import Workflow

```bash
# 1. Write import blocks in imports.tf
# 2. Run plan with generate-config-out (optional)
tofu plan -generate-config-out=generated.tf

# 3. Review generated.tf and clean up computed attributes
# 4. Run apply to import
tofu apply

# 5. Verify import - plan should show no changes
tofu plan

# 6. Remove import blocks (they are one-time operations)
# 7. Commit configuration and state
```

## Summary

Import blocks are the modern, declarative way to import existing cloud resources. Use `for_each` on import blocks for bulk imports, `-generate-config-out` to auto-generate HCL configuration, and always run `tofu plan` after importing to verify no unexpected changes. Remove import blocks after successful import - they're only needed for the initial import operation. The CLI `tofu import` command is still available for quick one-off imports but import blocks are preferred for reproducible workflows.
