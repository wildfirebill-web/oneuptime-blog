# How to Deploy Apache Kafka on AWS MSK with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, MSK, Kafka, Infrastructure as Code

Description: Learn how to deploy Apache Kafka on AWS Managed Streaming for Apache Kafka (MSK) with OpenTofu for production-grade managed Kafka clusters.

AWS MSK is a managed Kafka service that handles broker provisioning, patching, and scaling. Managing MSK clusters in OpenTofu ensures consistent broker configuration, networking, and security settings are deployed and version-controlled.

## Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

## MSK Cluster

```hcl
resource "aws_msk_cluster" "main" {
  cluster_name           = "production-kafka"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3  # Must be a multiple of AZs

  broker_node_group_info {
    instance_type   = "kafka.m5.large"
    client_subnets  = [
      aws_subnet.kafka_a.id,
      aws_subnet.kafka_b.id,
      aws_subnet.kafka_c.id,
    ]
    security_groups = [aws_security_group.kafka.id]

    storage_info {
      ebs_storage_info {
        volume_size = 1000  # GB per broker
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"          # TLS, TLS_PLAINTEXT, PLAINTEXT
      in_cluster    = true
    }
    encryption_at_rest_kms_key_arn = aws_kms_key.kafka.arn
  }

  client_authentication {
    sasl {
      iam  = true   # Use IAM for authentication
    }
  }

  configuration_info {
    arn      = aws_msk_configuration.main.arn
    revision = aws_msk_configuration.main.latest_revision
  }

  open_monitoring {
    prometheus {
      jmx_exporter  { enabled_in_broker = true }
      node_exporter { enabled_in_broker = true }
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.kafka.name
      }
      s3 {
        enabled = true
        bucket  = aws_s3_bucket.kafka_logs.id
        prefix  = "kafka-logs/"
      }
    }
  }

  tags = {
    Environment = "production"
    Team        = "data-platform"
  }
}
```

## MSK Configuration

```hcl
resource "aws_msk_configuration" "main" {
  name              = "production-kafka-config"
  kafka_versions    = ["3.5.1"]

  server_properties = <<EOT
auto.create.topics.enable=false
delete.topic.enable=true
log.retention.hours=168
log.retention.bytes=107374182400
min.insync.replicas=2
num.partitions=6
default.replication.factor=3
EOT
}
```

## Security Group

```hcl
resource "aws_security_group" "kafka" {
  name        = "kafka-sg"
  description = "Security group for MSK cluster"
  vpc_id      = aws_vpc.main.id

  # Kafka TLS port
  ingress {
    from_port       = 9094
    to_port         = 9094
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # Zookeeper (if needed for older clients)
  ingress {
    from_port       = 2181
    to_port         = 2181
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

## IAM Policy for Producer

```hcl
resource "aws_iam_role_policy" "kafka_producer" {
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kafka-cluster:Connect",
          "kafka-cluster:DescribeTopic",
          "kafka-cluster:WriteData",
        ]
        Resource = [
          aws_msk_cluster.main.arn,
          "arn:aws:kafka:${var.region}:${data.aws_caller_identity.current.account_id}:topic/${aws_msk_cluster.main.cluster_name}/*",
        ]
      }
    ]
  })
}
```

## Outputs

```hcl
output "bootstrap_brokers_tls" {
  value     = aws_msk_cluster.main.bootstrap_brokers_tls
  sensitive = true
}
```

## Conclusion

AWS MSK clusters in OpenTofu provide production-ready managed Kafka without operational overhead. Configure MSK with TLS-only client connections, IAM authentication, custom broker configuration for retention and replication, and enable Prometheus metrics and CloudWatch logging. Set min.insync.replicas=2 and default.replication.factor=3 for durability in a 3-broker cluster.
