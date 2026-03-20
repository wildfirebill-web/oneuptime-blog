# How to Deploy Multi-AZ RDS Databases with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Multi-AZ, High Availability, Infrastructure as Code

Description: Learn how to configure AWS RDS Multi-AZ deployments with OpenTofu for automatic failover and high availability.

---

RDS Multi-AZ deployments maintain a synchronous standby replica in a different Availability Zone. If the primary instance fails, RDS automatically fails over to the standby with minimal downtime — typically 1-2 minutes.

---

## Enable Multi-AZ on an RDS Instance

```hcl
resource "aws_db_instance" "main" {
  identifier        = "main-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  storage_type      = "gp3"

  db_name  = "appdb"
  username = "dbadmin"
  password = var.db_password

  # Enable Multi-AZ
  multi_az = true

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:05:00-Mon:06:00"

  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "main-db-final"

  tags = {
    Name = "main-db-multi-az"
  }
}
```

---

## Multi-AZ vs. Read Replicas

| Feature          | Multi-AZ                    | Read Replica              |
|------------------|-----------------------------|---------------------------|
| Purpose          | High availability/failover  | Read scaling              |
| Synchronization  | Synchronous                 | Asynchronous              |
| Accessible       | No (standby only)           | Yes (for reads)           |
| Automatic failover | Yes                       | No                        |
| Additional cost  | ~2x instance cost           | ~1x additional instance   |

---

## DB Subnet Group Across AZs

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id  # Subnets in multiple AZs

  tags = {
    Name = "main-db-subnet-group"
  }
}
```

The subnet group must include subnets in at least 2 AZs for Multi-AZ to work.

---

## Monitoring Failover Events

```hcl
resource "aws_cloudwatch_metric_alarm" "rds_failover" {
  alarm_name          = "rds-failover"
  metric_name         = "RDSFailoverCount"
  namespace           = "AWS/RDS"
  period              = 300
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  statistic           = "Sum"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## Test a Failover

```bash
# Trigger a manual failover (for testing)
aws rds reboot-db-instance   --db-instance-identifier main-db   --force-failover

# Monitor events
aws rds describe-events   --source-identifier main-db   --source-type db-instance   --duration 60
```

---

## Summary

Set `multi_az = true` on `aws_db_instance` to enable synchronous standby replication in a second AZ. Ensure the `aws_db_subnet_group` includes subnets in at least two AZs. The endpoint address stays the same during failover — RDS updates the DNS record to point to the new primary. Monitor failovers with CloudWatch Events and test periodically with a manual reboot failover.
