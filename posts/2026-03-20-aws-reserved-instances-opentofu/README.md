# How to Manage AWS Reserved Instances with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Reserved Instances, Cost Optimization, Infrastructure as Code

Description: Learn how to purchase and manage AWS Reserved Instances with OpenTofu for RDS, ElastiCache, and other services to reduce costs with 1 or 3-year commitments.

Reserved Instances (RIs) provide discounts up to 75% compared to On-Demand pricing for RDS, ElastiCache, Redshift, and OpenSearch in exchange for usage commitments. EC2 compute is better served by Savings Plans, but database services still use RIs.

## RDS Reserved Instances

```hcl
resource "aws_db_instance" "main" {
  identifier        = "production-db"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r7g.large"
  allocated_storage = 100

  # ... other settings ...
}

# Purchase a reserved instance for the RDS class
resource "aws_rds_reserved_instance" "main" {
  reserved_instance_id     = "production-rds-ri"
  offering_id              = data.aws_rds_reserved_instance_offering.main.offering_id
  instance_count           = 1

  tags = {
    Purpose = "Production RDS cost reduction"
  }
}

data "aws_rds_reserved_instance_offering" "main" {
  db_instance_class   = "db.r7g.large"
  duration            = 31536000  # 1 year in seconds
  multi_az            = false
  offering_type       = "No Upfront"
  product_description = "postgresql"
}
```

## ElastiCache Reserved Nodes

```hcl
data "aws_elasticache_reserved_cache_node_offering" "redis" {
  cache_node_type   = "cache.r7g.large"
  duration          = "1yr"
  offering_type     = "No Upfront"
  product_description = "redis"
}

resource "aws_elasticache_reserved_cache_node" "redis" {
  reserved_cache_node_id     = "production-redis-ri"
  reserved_cache_nodes_offering_id = data.aws_elasticache_reserved_cache_node_offering.redis.offering_id
  cache_node_count           = 2  # 1 primary + 1 replica

  tags = {
    Purpose = "Production Redis cost reduction"
  }
}
```

## OpenSearch Reserved Instances

```hcl
resource "aws_opensearch_domain" "main" {
  domain_name    = "production-search"
  engine_version = "OpenSearch_2.11"

  cluster_config {
    instance_type  = "r6g.large.search"
    instance_count = 3
  }
  # ... other settings ...
}

# Purchase reserved instances for OpenSearch
resource "aws_opensearch_reserved_instance" "main" {
  reserved_instance_id = "production-opensearch-ri"
  offering_id          = data.aws_opensearch_reserved_instance_offering.main.offering_id
  instance_count       = 3

  tags = {
    Purpose = "Production OpenSearch cost reduction"
  }
}

data "aws_opensearch_reserved_instance_offering" "main" {
  instance_type   = "r6g.large.search"
  duration        = 31536000
  payment_option  = "NO_UPFRONT"
}
```

## Capacity Reservations for EC2 (Guaranteed Capacity)

```hcl
# On-Demand Capacity Reservation — reserves capacity without a pricing commitment
resource "aws_ec2_capacity_reservation" "app" {
  instance_type     = "r7g.large"
  instance_platform = "Linux/UNIX"
  availability_zone = "us-east-1a"
  instance_count    = 10

  instance_match_criteria = "open"  # Any instance in the AZ can use it
  tenancy                 = "default"

  tags = {
    Purpose = "Reserved capacity for production auto-scaling"
  }
}
```

## Budget Alert for RI Utilization

```hcl
resource "aws_budgets_budget" "ri_utilization" {
  name         = "reserved-instance-utilization"
  budget_type  = "RI_UTILIZATION"
  limit_amount = "100"
  limit_unit   = "PERCENTAGE"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Service"
    values = ["Amazon Relational Database Service"]
  }

  notification {
    comparison_operator        = "LESS_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finops@example.com"]
  }
}
```

## Conclusion

AWS Reserved Instances in OpenTofu give you cost-optimized database and search infrastructure. Use RIs for RDS, ElastiCache, and OpenSearch where Savings Plans don't apply, match the reserved instance class exactly to your running instances, and monitor utilization to ensure you're not paying for idle reservations. Set budget alerts to catch underutilization before it becomes a cost leak.
