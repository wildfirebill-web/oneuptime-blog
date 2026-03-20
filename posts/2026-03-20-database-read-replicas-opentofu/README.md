# How to Create RDS Read Replicas with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Read Replicas, Database, High Availability

Description: Learn how to provision AWS RDS read replicas using OpenTofu to scale read traffic and reduce primary database load.

---

RDS read replicas are asynchronous copies of your primary database that serve read traffic. OpenTofu lets you declare replicas and their configuration as code, making it easy to scale read capacity.

---

## Create a Read Replica

```hcl
# Primary database (must have backups enabled)
resource "aws_db_instance" "primary" {
  identifier              = "primary-db"
  engine                  = "postgres"
  engine_version          = "15.4"
  instance_class          = "db.t3.medium"
  allocated_storage       = 100
  backup_retention_period = 7  # Required for replicas

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  skip_final_snapshot    = false
  final_snapshot_identifier = "primary-db-final"
}

# Read replica
resource "aws_db_instance" "replica" {
  identifier          = "read-replica-1"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.t3.medium"

  # No db_name, username, password needed — inherited from primary
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.rds.id]
  skip_final_snapshot    = true

  tags = {
    Name = "read-replica-1"
  }
}
```

---

## Cross-Region Read Replica

```hcl
provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_db_instance" "eu_replica" {
  provider            = aws.eu_west
  identifier          = "read-replica-eu"
  replicate_source_db = aws_db_instance.primary.arn  # Use ARN for cross-region
  instance_class      = "db.t3.medium"
  skip_final_snapshot = true
}
```

---

## Promote a Replica to Standalone

```bash
# Promote the replica (breaks replication)
aws rds promote-read-replica   --db-instance-identifier read-replica-1

# Verify it's standalone
aws rds describe-db-instances   --db-instance-identifier read-replica-1   --query 'DBInstances[*].ReadReplicaSourceDBInstanceIdentifier'
```

---

## Output Replica Endpoint

```hcl
output "replica_endpoint" {
  value = aws_db_instance.replica.endpoint
}
```

Configure your application to send reads to the replica endpoint and writes to the primary endpoint.

---

## Summary

Create read replicas by specifying `replicate_source_db` pointing to the primary instance's identifier. The primary must have `backup_retention_period > 0`. For cross-region replicas, use the primary's ARN instead of identifier. Replicas inherit the engine, storage, and authentication from the primary. Promote a replica to standalone with `aws rds promote-read-replica` when needed.
