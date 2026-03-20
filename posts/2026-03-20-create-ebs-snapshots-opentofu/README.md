# How to Create EBS Snapshots with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EBS, Snapshot, Infrastructure as Code

Description: Learn how to automate EBS snapshot creation with OpenTofu using DLM lifecycle policies for consistent, cost-effective disk backup.

EBS snapshots capture point-in-time copies of volumes for backup and disaster recovery. AWS Data Lifecycle Manager (DLM) automates snapshot creation and retention. Managing DLM policies in OpenTofu ensures consistent snapshot schedules across all volumes.

## DLM Lifecycle Policy

```hcl
resource "aws_dlm_lifecycle_policy" "ebs" {
  description        = "Daily EBS snapshot policy"
  execution_role_arn = aws_iam_role.dlm.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    schedule {
      name = "daily-snapshots"

      create_rule {
        interval      = 24
        interval_unit = "HOURS"
        times         = ["03:00"]  # UTC time
      }

      retain_rule {
        count = 30  # Keep last 30 snapshots
      }

      tags_to_add = {
        SnapshotCreator = "DLM"
        BackupPolicy    = "daily"
      }

      copy_tags = true  # Copy tags from source volume

      # Cross-region copy
      cross_region_copy_rule {
        target    = "us-west-2"
        encrypted = true
        copy_tags = true

        retain_rule {
          interval      = 30
          interval_unit = "DAYS"
        }
      }
    }

    # Target volumes by tag
    target_tags = {
      Backup = "true"
    }
  }

  tags = {
    Purpose = "EBS snapshot automation"
  }
}
```

## IAM Role for DLM

```hcl
resource "aws_iam_role" "dlm" {
  name = "dlm-lifecycle-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "dlm.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "dlm" {
  role       = aws_iam_role.dlm.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole"
}
```

## Multi-Schedule Policy

```hcl
resource "aws_dlm_lifecycle_policy" "multi_schedule" {
  description        = "Tiered EBS snapshot policy"
  execution_role_arn = aws_iam_role.dlm.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    # Hourly snapshots for critical volumes - keep 24
    schedule {
      name = "hourly"

      create_rule {
        interval      = 1
        interval_unit = "HOURS"
      }

      retain_rule {
        count = 24
      }

      tags_to_add = {
        SnapshotType = "hourly"
      }
    }

    # Weekly snapshots - keep 12
    schedule {
      name = "weekly"

      create_rule {
        cron_expression = "cron(0 2 ? * 7 *)"  # Saturdays at 2 AM
      }

      retain_rule {
        count = 12
      }

      tags_to_add = {
        SnapshotType = "weekly"
      }
    }

    target_tags = {
      TieredBackup = "true"
    }
  }
}
```

## Tagging EBS Volumes for Backup

```hcl
# Tag volumes for automatic inclusion in DLM policy

resource "aws_ebs_volume" "app_data" {
  availability_zone = "us-east-1a"
  size              = 100
  type              = "gp3"

  tags = {
    Name         = "app-data"
    Environment  = "production"
    Backup       = "true"  # Picked up by DLM policy
    TieredBackup = "true"
  }
}
```

## Manual Snapshot for AMI Creation

```hcl
# One-time snapshot for AMI creation before major changes
resource "aws_ebs_snapshot" "pre_migration" {
  volume_id   = aws_ebs_volume.app_data.id
  description = "Pre-migration snapshot ${formatdate("YYYY-MM-DD", timestamp())}"

  tags = {
    Name    = "pre-migration-snapshot"
    Reason  = "manual-before-migration"
  }
}
```

## Conclusion

EBS snapshots automated via DLM in OpenTofu provide consistent, cost-effective disk backup. Tag volumes with Backup=true for automatic inclusion in policies, configure cross-region copy for DR resilience, and use multi-schedule policies for tiered retention. DLM manages the full lifecycle including creation, retention, and deletion, reducing storage costs by automatically cleaning up old snapshots.
