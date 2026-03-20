# How to Create an RDS Database with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Database, Infrastructure as Code, PostgreSQL

Description: Learn how to provision an AWS RDS database instance with subnet groups, parameter groups, and security groups using OpenTofu.

---

AWS RDS provides managed relational databases. OpenTofu automates provisioning the entire RDS stack - subnet groups, parameter groups, security groups, and the database instance - ensuring consistent, repeatable deployments.

---

## Create a DB Subnet Group

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "main-db-subnet-group"
  }
}
```

---

## Create a Security Group for RDS

```hcl
resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

---

## Create the RDS Instance

```hcl
resource "aws_db_instance" "main" {
  identifier        = "main-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  storage_type      = "gp3"

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = false
  publicly_accessible    = false
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "main-db-final-snapshot"

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  tags = {
    Name = "main-db"
  }
}
```

---

## Output the Connection Details

```hcl
output "db_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "db_name" {
  value = aws_db_instance.main.db_name
}
```

---

## Enable Enhanced Monitoring

```hcl
resource "aws_db_instance" "main" {
  # ... other settings ...
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}
```

---

## Summary

Create `aws_db_subnet_group` to place the database in private subnets, a security group allowing access only from application instances, and `aws_db_instance` with engine, storage, credentials, and backup settings. Enable `deletion_protection` and configure `backup_retention_period` for production databases. Always use `skip_final_snapshot = false` in production.
