# How to Migrate Databases Between Regions with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Database Migration, Multi-Region, Infrastructure as Code

Description: Learn how to migrate RDS databases between AWS regions using snapshots, cross-region replication, and read replicas with OpenTofu.

## Introduction

Migrating a database to a different region requires careful planning to minimize downtime and ensure data consistency. OpenTofu orchestrates the creation of cross-region snapshots, replica instances, and the final cutover.

## Strategy 1: Cross-Region Snapshot Copy

```hcl
# Source region (us-east-1)
provider "aws" {
  alias  = "source"
  region = "us-east-1"
}

# Target region (eu-west-1)
provider "aws" {
  alias  = "target"
  region = "eu-west-1"
}

# Create a final snapshot in the source region
resource "aws_db_snapshot" "migration_snapshot" {
  provider               = aws.source
  db_instance_identifier = var.source_db_identifier
  db_snapshot_identifier = "${var.app_name}-migration-${formatdate("YYYYMMDDhhmmss", timestamp())}"
}

# Copy snapshot to the target region
resource "aws_db_snapshot_copy" "to_target_region" {
  provider               = aws.target
  source_db_snapshot_identifier = aws_db_snapshot.migration_snapshot.db_snapshot_arn
  source_region          = "us-east-1"
  target_db_snapshot_identifier = "${var.app_name}-migration-target"

  # Encrypt in target region with target region KMS key
  kms_key_id = aws_kms_key.target_db.arn

  tags = {
    Purpose   = "region-migration"
    ManagedBy = "opentofu"
  }
}
```

## Restoring from the Copied Snapshot

```hcl
# Restore the database in the target region from the copied snapshot
resource "aws_db_instance" "migrated" {
  provider               = aws.target
  identifier             = "${var.app_name}-db-${var.target_environment}"
  snapshot_identifier    = aws_db_snapshot_copy.to_target_region.id

  instance_class         = var.db_instance_class
  db_subnet_group_name   = aws_db_subnet_group.target.name
  vpc_security_group_ids = [aws_security_group.db_target.id]

  # Override encryption with target region key
  storage_encrypted  = true
  kms_key_id         = aws_kms_key.target_db.arn

  apply_immediately   = true
  skip_final_snapshot = false
  final_snapshot_identifier = "${var.app_name}-db-final"

  tags = {
    Environment = var.target_environment
    ManagedBy   = "opentofu"
  }
}
```

## Strategy 2: Cross-Region Read Replica

For lower RPO/RTO, promote a cross-region read replica.

```hcl
# Create a cross-region read replica for near-zero RPO
resource "aws_db_instance" "cross_region_replica" {
  provider = aws.target

  identifier            = "${var.app_name}-db-replica-${var.target_environment}"
  replicate_source_db   = var.source_db_arn  # ARN of source DB

  instance_class         = var.db_instance_class
  db_subnet_group_name   = aws_db_subnet_group.target.name
  vpc_security_group_ids = [aws_security_group.db_target.id]

  # Replicas inherit encryption from source; specify target key
  kms_key_id = aws_kms_key.target_db.arn

  backup_retention_period = 0  # disable backups on replica
  skip_final_snapshot     = true
  apply_immediately       = true
}
```

## Promoting the Replica

When ready to cut over, promote the replica to a standalone instance.

```bash
# Stop writes to the source database (application-level)
# Wait for replica lag to reach 0

# Promote the read replica
aws rds promote-read-replica \
  --db-instance-identifier "${APP_NAME}-db-replica-${TARGET_ENV}" \
  --region eu-west-1

# Update application connection strings to point to new DB
tofu apply -var="db_endpoint=${NEW_DB_ENDPOINT}"
```

## Target Region Networking

```hcl
resource "aws_db_subnet_group" "target" {
  provider   = aws.target
  name       = "${var.app_name}-db-subnets-target"
  subnet_ids = var.target_private_subnet_ids
}

resource "aws_kms_key" "target_db" {
  provider    = aws.target
  description = "KMS key for ${var.app_name} database in target region"
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Database region migration requires a choice between snapshot-based migration (higher RPO) and cross-region read replica promotion (near-zero RPO). OpenTofu manages snapshot copies, restored instances, cross-region replicas, and target networking — providing a reproducible migration process with minimal manual steps.
