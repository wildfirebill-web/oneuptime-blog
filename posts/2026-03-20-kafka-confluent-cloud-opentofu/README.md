# How to Deploy Kafka on Confluent Cloud with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Confluent Cloud, Kafka, Streaming, Infrastructure as Code, Data Engineering, Managed Service

Description: Learn how to provision Confluent Cloud Kafka clusters, topics, service accounts, and API keys using OpenTofu for fully managed, serverless Kafka deployments.

---

Confluent Cloud is the fully managed Kafka service that handles all operational complexity. With OpenTofu's Confluent provider, you can define clusters, topics, ACLs, and service accounts as code - making your streaming infrastructure as reproducible as your application infrastructure.

## Provider Configuration

```hcl
# main.tf

terraform {
  required_providers {
    confluent = {
      source  = "confluentinc/confluent"
      version = "~> 1.76"
    }
  }
}

provider "confluent" {
  cloud_api_key    = var.confluent_cloud_api_key
  cloud_api_secret = var.confluent_cloud_api_secret
}
```

## Creating an Environment and Cluster

```hcl
# cluster.tf
# Create a Confluent environment (logical grouping for clusters)
resource "confluent_environment" "production" {
  display_name = "production"

  stream_governance {
    package = "ESSENTIALS"
  }
}

# Create a Kafka cluster
resource "confluent_kafka_cluster" "main" {
  display_name = "production-kafka"
  availability = "MULTI_ZONE"  # Multi-zone for HA
  cloud        = var.cloud_provider  # AWS, GCP, or AZURE
  region       = var.cloud_region

  # Choose cluster type based on your throughput needs
  # basic {} for development/testing
  # standard {} for production
  # dedicated { cku = 2 } for high-throughput production
  standard {}

  environment {
    id = confluent_environment.production.id
  }
}
```

## Creating Topics

```hcl
# topics.tf
# Service account for the topic admin role
resource "confluent_service_account" "topic_admin" {
  display_name = "topic-admin"
  description  = "Service account for managing topics"
}

resource "confluent_role_binding" "topic_admin" {
  principal   = "User:${confluent_service_account.topic_admin.id}"
  role_name   = "CloudClusterAdmin"
  crn_pattern = confluent_kafka_cluster.main.rbac_crn
}

resource "confluent_api_key" "topic_admin" {
  display_name = "topic-admin-api-key"
  owner {
    id          = confluent_service_account.topic_admin.id
    api_version = confluent_service_account.topic_admin.api_version
    kind        = confluent_service_account.topic_admin.kind
  }
  managed_resource {
    id          = confluent_kafka_cluster.main.id
    api_version = confluent_kafka_cluster.main.api_version
    kind        = confluent_kafka_cluster.main.kind
    environment {
      id = confluent_environment.production.id
    }
  }
}

# Create application topics
variable "topics" {
  type = map(object({
    partitions        = number
    retention_ms      = string
    cleanup_policy    = string
  }))
  default = {
    "orders" = {
      partitions     = 6
      retention_ms   = "604800000"  # 7 days
      cleanup_policy = "delete"
    }
    "user-events" = {
      partitions     = 12
      retention_ms   = "86400000"   # 1 day
      cleanup_policy = "delete"
    }
  }
}

resource "confluent_kafka_topic" "topics" {
  for_each = var.topics

  kafka_cluster {
    id = confluent_kafka_cluster.main.id
  }

  topic_name       = each.key
  partitions_count = each.value.partitions

  config = {
    "retention.ms"    = each.value.retention_ms
    "cleanup.policy"  = each.value.cleanup_policy
    "min.insync.replicas" = "2"
  }

  rest_endpoint = confluent_kafka_cluster.main.rest_endpoint
  credentials {
    key    = confluent_api_key.topic_admin.id
    secret = confluent_api_key.topic_admin.secret
  }
}
```

## Creating Service Accounts with ACLs

```hcl
# service_accounts.tf
# Producer service account
resource "confluent_service_account" "orders_producer" {
  display_name = "orders-producer"
  description  = "Service account for the orders service producer"
}

resource "confluent_api_key" "orders_producer" {
  display_name = "orders-producer-key"
  owner {
    id          = confluent_service_account.orders_producer.id
    api_version = confluent_service_account.orders_producer.api_version
    kind        = confluent_service_account.orders_producer.kind
  }
  managed_resource {
    id          = confluent_kafka_cluster.main.id
    api_version = confluent_kafka_cluster.main.api_version
    kind        = confluent_kafka_cluster.main.kind
    environment {
      id = confluent_environment.production.id
    }
  }
}

# Grant the producer WRITE access to the orders topic only
resource "confluent_kafka_acl" "orders_producer_write" {
  kafka_cluster {
    id = confluent_kafka_cluster.main.id
  }

  resource_type = "TOPIC"
  resource_name = "orders"
  pattern_type  = "LITERAL"
  principal     = "User:${confluent_service_account.orders_producer.id}"
  host          = "*"
  operation     = "WRITE"
  permission    = "ALLOW"

  rest_endpoint = confluent_kafka_cluster.main.rest_endpoint
  credentials {
    key    = confluent_api_key.topic_admin.id
    secret = confluent_api_key.topic_admin.secret
  }
}
```

## Best Practices

- Use `MULTI_ZONE` availability for production clusters to ensure broker failures don't cause downtime.
- Use service accounts and API keys per application rather than sharing credentials - this enables per-application ACL management.
- Set `min.insync.replicas=2` on all topics to prevent data loss during broker failures.
- Use topic naming conventions (`<domain>.<entity>.<event>`) to make schemas self-documenting.
- Enable Schema Registry to enforce Avro/Protobuf schemas and prevent incompatible producer changes from breaking consumers.
