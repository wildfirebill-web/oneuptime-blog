# How to Create AWS OpenSearch Domains with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, OpenSearch, Search, Infrastructure as Code

Description: Learn how to create AWS OpenSearch Service domains with OpenTofu for managed full-text search, log analytics, and observability workloads.

AWS OpenSearch Service (formerly Elasticsearch) provides managed search and analytics. Managing domains in OpenTofu ensures consistent instance type, storage, authentication, and encryption settings across environments.

## Creating an OpenSearch Domain

```hcl
resource "aws_opensearch_domain" "main" {
  domain_name    = "myapp-search"
  engine_version = "OpenSearch_2.11"

  cluster_config {
    instance_type          = "r6g.large.search"
    instance_count         = 3  # Minimum 3 for HA

    dedicated_master_enabled = true
    dedicated_master_type    = "r6g.large.search"
    dedicated_master_count   = 3

    zone_awareness_enabled = true
    zone_awareness_config {
      availability_zone_count = 3
    }

    warm_enabled = false  # Enable for UltraWarm storage tier
  }

  ebs_options {
    ebs_enabled = true
    volume_size = 100   # GB per node
    volume_type = "gp3"
    throughput  = 250   # MB/s
    iops        = 3000
  }

  encrypt_at_rest {
    enabled    = true
    kms_key_id = aws_kms_key.opensearch.arn
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  advanced_security_options {
    enabled                        = true
    anonymous_auth_enabled         = false
    internal_user_database_enabled = false

    master_user_options {
      master_user_arn = aws_iam_role.opensearch_admin.arn
    }
  }

  vpc_options {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.opensearch.id]
  }

  log_publishing_options {
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_index.arn
    log_type                 = "INDEX_SLOW_LOGS"
  }

  log_publishing_options {
    cloudwatch_log_group_arn = aws_cloudwatch_log_group.opensearch_search.arn
    log_type                 = "SEARCH_SLOW_LOGS"
  }

  tags = {
    Environment = "production"
    Team        = "data-platform"
  }
}
```

## Access Policy

```hcl
resource "aws_opensearch_domain_policy" "main" {
  domain_name = aws_opensearch_domain.main.domain_name

  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { AWS = aws_iam_role.app_role.arn }
        Action    = ["es:ESHttpGet", "es:ESHttpPost", "es:ESHttpPut", "es:ESHttpDelete"]
        Resource  = "${aws_opensearch_domain.main.arn}/*"
      }
    ]
  })
}
```

## Security Group

```hcl
resource "aws_security_group" "opensearch" {
  name        = "opensearch-sg"
  description = "OpenSearch domain security group"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }
}
```

## CloudWatch Alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "cluster_status" {
  alarm_name          = "opensearch-cluster-red"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ClusterStatus.red"
  namespace           = "AWS/ES"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "OpenSearch cluster is in RED status"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    DomainName = aws_opensearch_domain.main.domain_name
    ClientId   = data.aws_caller_identity.current.account_id
  }
}
```

## Outputs

```hcl
output "opensearch_endpoint" {
  value = aws_opensearch_domain.main.endpoint
}

output "opensearch_dashboard_endpoint" {
  value = aws_opensearch_domain.main.dashboard_endpoint
}
```

## Conclusion

AWS OpenSearch domains in OpenTofu provide production-ready managed search. Deploy with 3 dedicated master nodes and 3 data nodes across AZs for HA, enable encryption at rest and in transit, configure fine-grained access control with IAM authentication, and set up slow log publishing to identify query performance issues.
