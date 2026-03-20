# How to Deploy Aurora Clusters with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Aurora, RDS, Database, Infrastructure as Code

Description: Learn how to provision an Amazon Aurora cluster with writer and reader instances using OpenTofu for high-performance managed databases.

---

Amazon Aurora is a MySQL and PostgreSQL-compatible relational database with up to 5x better performance than MySQL. It separates compute from storage and supports automatic failover across multiple replicas. OpenTofu manages the entire cluster configuration.

---

## Create an Aurora Cluster

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier      = "aurora-cluster"
  engine                  = "aurora-postgresql"
  engine_version          = "15.4"
  availability_zones      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  database_name           = "appdb"
  master_username         = "dbadmin"
  master_password         = var.db_password
  db_subnet_group_name    = aws_db_subnet_group.aurora.name
  vpc_security_group_ids  = [aws_security_group.aurora.id]
  backup_retention_period = 7
  preferred_backup_window = "07:00-09:00"
  skip_final_snapshot     = false
  final_snapshot_identifier = "aurora-final-snapshot"
  deletion_protection     = true

  tags = {
    Name = "aurora-production"
  }
}
```

---

## Add Cluster Instances (Writer + Readers)

```hcl
# Writer instance
resource "aws_rds_cluster_instance" "writer" {
  identifier         = "aurora-writer"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version
}

# Reader instances
resource "aws_rds_cluster_instance" "readers" {
  count              = 2
  identifier         = "aurora-reader-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aurora.engine
  engine_version     = aws_rds_cluster.aurora.engine_version
}
```

---

## Aurora Serverless v2

```hcl
resource "aws_rds_cluster" "serverless" {
  cluster_identifier = "aurora-serverless"
  engine             = "aurora-postgresql"
  engine_version     = "15.4"
  engine_mode        = "provisioned"  # Serverless v2 uses provisioned mode
  database_name      = "appdb"
  master_username    = "dbadmin"
  master_password    = var.db_password

  serverlessv2_scaling_configuration {
    min_capacity = 0.5  # ACU
    max_capacity = 16.0
  }
}

resource "aws_rds_cluster_instance" "serverless" {
  cluster_identifier = aws_rds_cluster.serverless.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.serverless.engine
}
```

---

## Output Endpoints

```hcl
output "writer_endpoint" {
  value = aws_rds_cluster.aurora.endpoint
}

output "reader_endpoint" {
  value = aws_rds_cluster.aurora.reader_endpoint
}
```

---

## Summary

Create `aws_rds_cluster` for the Aurora cluster definition, then add `aws_rds_cluster_instance` resources for writer and reader instances. Use the cluster's `endpoint` for write traffic and `reader_endpoint` for read traffic — Aurora automatically routes to the current writer and distributes reads. For variable workloads, use Serverless v2 with `instance_class = "db.serverless"`.
