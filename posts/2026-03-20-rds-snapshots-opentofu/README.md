# How to Create RDS Snapshots with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Snapshots, Backup, Disaster Recovery, Infrastructure as Code

Description: Learn how to create manual RDS snapshots and manage snapshot copies across regions using OpenTofu for long-term backup retention and cross-region disaster recovery.

## Introduction

Manual RDS snapshots are user-initiated backups retained until explicitly deleted, unlike automated backups that expire after the retention window. They are used for long-term retention, pre-change backups, and cross-region copies for disaster recovery.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with RDS permissions

## Step 1: Create a Manual Snapshot

```hcl
# Manual snapshot of an RDS instance
resource "aws_db_snapshot" "pre_migration" {
  db_instance_identifier = aws_db_instance.main.id
  db_snapshot_identifier = "${var.project_name}-pre-migration-${formatdate("YYYYMMDD", timestamp())}"

  tags = {
    Name    = "pre-migration-snapshot"
    Reason  = "PreMigration"
    Version = var.app_version
  }

  # Timeouts for snapshot creation
  timeouts {
    create = "30m"
  }
}
```

## Step 2: Copy Snapshot to Another Region

```hcl
# Copy the snapshot to a DR region for cross-region recovery
resource "aws_db_snapshot_copy" "dr_copy" {
  provider = aws.dr_region

  source_db_snapshot_identifier = aws_db_snapshot.pre_migration.db_snapshot_arn
  target_db_snapshot_identifier = "${var.project_name}-dr-${formatdate("YYYYMMDD", timestamp())}"
  source_region                 = var.primary_region

  # Encrypt the copy with the DR region's KMS key
  kms_key_id = var.dr_kms_key_arn
  copy_tags  = true

  tags = {
    Name         = "dr-region-snapshot"
    SourceRegion = var.primary_region
  }
}
```

## Step 3: Share Snapshot with Another Account

```hcl
# Share a snapshot with a DR or audit account
resource "aws_db_snapshot" "shared" {
  db_instance_identifier = aws_db_instance.main.id
  db_snapshot_identifier = "${var.project_name}-shared-${formatdate("YYYYMMDD", timestamp())}"
}

# Modify snapshot attributes to share with specific accounts
resource "null_resource" "share_snapshot" {
  triggers = {
    snapshot_id = aws_db_snapshot.shared.id
  }

  provisioner "local-exec" {
    command = <<-EOF
      aws rds modify-db-snapshot-attribute \
        --db-snapshot-identifier ${aws_db_snapshot.shared.id} \
        --attribute-name restore \
        --values-to-add ${var.target_account_id} \
        --region ${var.region}
    EOF
  }
}
```

## Step 4: Restore from a Snapshot

```hcl
# Restore a new database instance from a snapshot
resource "aws_db_instance" "restored" {
  identifier     = "${var.project_name}-restored"
  instance_class = "db.r6g.xlarge"

  # Specify the snapshot to restore from
  snapshot_identifier = aws_db_snapshot.pre_migration.id

  # Override network settings for the restored instance
  db_subnet_group_name   = var.subnet_group_name
  vpc_security_group_ids = [var.security_group_id]

  # These settings are required even for snapshot restores
  skip_final_snapshot = true

  tags = {
    Name      = "restored-instance"
    Source    = "snapshot"
    SourceSnap = aws_db_snapshot.pre_migration.id
  }
}
```

## Step 5: Create a Snapshot Before Terraform Destroy

```hcl
# Final snapshot created automatically when the instance is deleted
resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-db"

  # Prevent accidental deletion
  deletion_protection = true

  # Create a final snapshot before deletion
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project_name}-final-snapshot"

  # Other configuration...
  engine         = "postgres"
  engine_version = "16.2"
  instance_class = "db.t3.medium"
  db_name        = var.database_name
  username       = var.master_username
  password       = var.master_password
  storage_type      = "gp3"
  allocated_storage = 20
  db_subnet_group_name   = var.subnet_group_name
  vpc_security_group_ids = [var.security_group_id]
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

Manual RDS snapshots provide a reliable mechanism for long-term backup retention beyond the automated backup window (max 35 days). Take manual snapshots before major changes like schema migrations or upgrades as a recovery safety net. Encrypt snapshots with customer-managed KMS keys for compliance, and copy snapshots to additional regions for geographic redundancy in your DR strategy.
