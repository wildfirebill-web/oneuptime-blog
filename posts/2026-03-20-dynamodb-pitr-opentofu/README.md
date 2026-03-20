# How to Enable DynamoDB Point-in-Time Recovery with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, DynamoDB, PITR, Backup, Disaster Recovery, Terraform

Description: Learn how to enable and manage DynamoDB Point-in-Time Recovery (PITR) using OpenTofu to protect your data with continuous backups and restore to any point within 35 days.

---

DynamoDB Point-in-Time Recovery (PITR) provides continuous backups of your DynamoDB table data. It allows you to restore your table to any point in time within the last 35 days, protecting against accidental writes, deletes, or application bugs.

---

## Enabling PITR with OpenTofu

### Basic Table with PITR

```hcl
# main.tf
resource "aws_dynamodb_table" "users" {
  name           = "users"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "userId"

  attribute {
    name = "userId"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Environment = "production"
    DataClass   = "sensitive"
    ManagedBy   = "opentofu"
  }
}
```

### Enable PITR on Existing Table

```hcl
resource "aws_dynamodb_table" "orders" {
  name         = "orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "orderId"

  attribute {
    name = "orderId"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }
}
```

---

## Verify PITR is Enabled

```bash
# Check PITR status
aws dynamodb describe-continuous-backups --table-name users \
  --query 'ContinuousBackupsDescription.PointInTimeRecoveryDescription'

# Expected output:
# {
#   "PointInTimeRecoveryStatus": "ENABLED",
#   "EarliestRestorableDateTime": "2026-02-13T...",
#   "LatestRestorableDateTime": "2026-03-20T..."
# }
```

---

## Restoring from PITR

PITR restores create a new table — they do not overwrite the source.

### Restore via AWS CLI

```bash
# Restore to latest restorable time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name users \
  --target-table-name users-restored-20260320 \
  --use-latest-restorable-time

# Restore to a specific time
aws dynamodb restore-table-to-point-in-time \
  --source-table-name users \
  --target-table-name users-restored-before-incident \
  --restore-date-time "2026-03-19T14:30:00Z"

# Monitor restore status
aws dynamodb describe-table --table-name users-restored-20260320 \
  --query 'Table.TableStatus'
```

---

## Combining PITR with On-Demand Backups

Use both PITR and scheduled on-demand backups for defense in depth:

```hcl
# PITR for continuous protection
resource "aws_dynamodb_table" "critical" {
  name         = "transactions"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "txId"

  attribute {
    name = "txId"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }
}

# Manual on-demand backup (snapshot)
resource "aws_dynamodb_backup" "weekly" {
  table_name  = aws_dynamodb_table.critical.name
  backup_name = "transactions-weekly-${formatdate("YYYYMMDD", timestamp())}"

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## CloudWatch Monitoring for Backup Status

```hcl
resource "aws_cloudwatch_metric_alarm" "pitr_status" {
  alarm_name          = "dynamodb-pitr-disabled"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "SuccessfulRequestLatency"
  namespace           = "AWS/DynamoDB"
  period              = 300
  statistic           = "SampleCount"
  threshold           = 1
  alarm_description   = "Alert if DynamoDB is not receiving requests (proxy for health check)"

  dimensions = {
    TableName = aws_dynamodb_table.critical.name
    Operation = "GetItem"
  }
}
```

---

## PITR with AWS Backup Service

For centralized backup management across services, use AWS Backup:

```hcl
resource "aws_backup_plan" "dynamodb" {
  name = "dynamodb-backup-plan"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 2 * * ? *)"  # 2 AM daily
    
    lifecycle {
      delete_after = 35
    }
  }
}

resource "aws_backup_selection" "dynamodb" {
  name         = "dynamodb-selection"
  plan_id      = aws_backup_plan.dynamodb.id
  iam_role_arn = aws_iam_role.backup.arn

  resources = [
    aws_dynamodb_table.critical.arn
  ]
}

resource "aws_backup_vault" "main" {
  name        = "dynamodb-backup-vault"
  kms_key_arn = aws_kms_key.backup.arn
}
```

---

## Cost Considerations

PITR pricing is based on table size:

| Factor | Cost |
|--------|------|
| PITR storage | ~$0.20 per GB per month |
| Data restore | Free |
| Cross-region restore | Standard data transfer rates |

---

## Best Practices

1. **Enable PITR on all production tables** — the cost is minimal compared to data loss risk
2. **Test restores regularly** — run quarterly restore drills to validate your recovery process
3. **Monitor earliest restorable time** — if it's older than 35 days, your window has expired
4. **Use separate restored table name** — never restore over an existing production table
5. **Combine with on-demand backups** for long-term archival beyond 35 days

---

## Conclusion

DynamoDB PITR is a one-line addition to any OpenTofu configuration that provides continuous protection with 35 days of recovery window. Enable it on all production tables, and pair it with AWS Backup for long-term archival.

---

*Monitor your database health and set up alerting with [OneUptime](https://oneuptime.com).*
