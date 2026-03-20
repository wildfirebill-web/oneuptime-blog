# How to Troubleshoot tofu apply Timeouts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Timeouts, Apply, Infrastructure as Code

Description: Learn how to diagnose and resolve tofu apply timeouts for resources that take longer than expected to create, update, or delete.

## Introduction

`tofu apply` timeouts occur when a cloud resource takes longer to reach the desired state than the provider's default timeout allows. Common offenders are RDS instances, EKS clusters, ElastiCache clusters, and OpenSearch domains. Understanding how to diagnose the cause and extend timeouts is essential for managing complex infrastructure.

## Identifying Timeout Errors

```
Error: waiting for RDS DB Instance creation: timeout while waiting for
state to become 'available' (last state: 'creating', timeout: 40m0s)
```

```bash
# Get more detail about what's happening
TF_LOG=INFO tofu apply 2>&1 | grep -i "waiting\|poll\|retry"

# Check the resource status directly in AWS
aws rds describe-db-instances \
  --db-instance-identifier myapp-prod-db \
  --query 'DBInstances[0].DBInstanceStatus'
```

## Solution 1: Extend Timeouts with timeouts Block

Most AWS provider resources support a `timeouts` block.

```hcl
resource "aws_db_instance" "main" {
  identifier    = "myapp-prod-db"
  engine        = "postgres"
  instance_class = "db.r6g.2xlarge"
  # Large instances take longer to create

  timeouts {
    create = "60m"  # default is 40m
    update = "80m"  # for major version upgrades
    delete = "60m"
  }
}

resource "aws_eks_cluster" "main" {
  name     = "myapp-cluster"
  # ...

  timeouts {
    create = "30m"
    update = "60m"
    delete = "15m"
  }
}

resource "aws_opensearch_domain" "main" {
  domain_name = "myapp-search"
  # ...

  timeouts {
    create = "90m"  # OpenSearch can take a very long time
    update = "60m"
    delete = "90m"
  }
}
```

## Solution 2: Check if Operation Is Actually Failing

Sometimes the timeout is a symptom, not the root cause.

```bash
# Check CloudTrail for API errors
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com \
  --start-time "2026-03-20T09:00:00" \
  --query 'Events[?ErrorCode!=null].[EventTime,ErrorCode,ErrorMessage]' \
  --output table

# Check resource events in the AWS console
aws rds describe-events \
  --source-identifier myapp-prod-db \
  --source-type db-instance \
  --duration 60  # last 60 minutes
```

## Solution 3: Handle Resource-Specific Timeout Causes

Different resources time out for different reasons.

```hcl
# RDS: parameter group changes can require reboot
resource "aws_db_parameter_group" "main" {
  family = "postgres15"
  # ...
}

resource "aws_db_instance" "main" {
  parameter_group_name = aws_db_parameter_group.main.name

  # Apply parameter changes after reboot cycle
  apply_immediately = false  # apply during maintenance window
}

# EKS: node groups may timeout if instance types are unavailable
resource "aws_eks_node_group" "app" {
  # Use multiple instance types for better availability
  instance_types = ["m5.xlarge", "m5a.xlarge", "m4.xlarge"]
  # ...

  timeouts {
    create = "60m"
    update = "60m"
  }
}

# VPC Peering: timeout waiting for acceptance
resource "aws_vpc_peering_connection_accepter" "peer" {
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id
  auto_accept               = true  # auto-accept prevents manual step timeout

  timeouts {
    create = "1m"
    delete = "1m"
  }
}
```

## Solution 4: Using -target for Problematic Resources

Apply problematic resources individually to avoid blocking the entire state.

```bash
# Apply the slow resource separately
tofu apply -target=aws_db_instance.main

# Then apply everything else after the database is ready
tofu apply -exclude=aws_db_instance.main
```

## Monitoring Long-Running Operations

```bash
# Watch resource status in another terminal while apply runs
watch -n 30 aws rds describe-db-instances \
  --db-instance-identifier myapp-prod-db \
  --query 'DBInstances[0].[DBInstanceStatus,PercentProgress]' \
  --output table

# For EKS cluster
watch -n 30 aws eks describe-cluster \
  --name myapp-cluster \
  --query 'cluster.status'
```

## Summary

Apply timeouts are usually caused by resource operations that legitimately take longer than the provider's default — extend them with the `timeouts` block. For RDS, EKS clusters, and OpenSearch domains, budget 60-90 minutes for creation timeouts. Always check CloudTrail and resource events to verify the timeout is not masking an underlying error (wrong subnet, missing permissions, quota exceeded). Use `-target` to apply slow resources individually so they don't block the rest of the configuration.
