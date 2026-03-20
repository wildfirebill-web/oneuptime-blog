# How to Deploy RabbitMQ with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, RabbitMQ, Messaging, Kubernetes, Helm, Infrastructure as Code, AMQP

Description: Learn how to deploy RabbitMQ on Kubernetes using OpenTofu with clustering, persistent storage, management UI, and federation for production message broker deployments.

---

RabbitMQ is the widely deployed open-source message broker that implements AMQP, MQTT, and STOMP. Deploying it on Kubernetes with the RabbitMQ Cluster Operator gives you automatic clustering, rolling updates, and declarative queue configuration. OpenTofu automates the entire setup.

## Deploying RabbitMQ with the Cluster Operator

```hcl
# main.tf
terraform {
  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.24"
    }
  }
}

provider "helm" {
  kubernetes {
    host                   = var.cluster_endpoint
    cluster_ca_certificate = base64decode(var.cluster_ca_cert)
    token                  = var.cluster_token
  }
}

resource "kubernetes_namespace" "rabbitmq" {
  metadata {
    name = "rabbitmq"
  }
}

# Deploy the RabbitMQ Cluster Operator
resource "helm_release" "rabbitmq_operator" {
  name       = "rabbitmq-cluster-operator"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "rabbitmq-cluster-operator"
  version    = "4.3.4"
  namespace  = kubernetes_namespace.rabbitmq.metadata[0].name

  wait    = true
  timeout = 300
}
```

## Creating a RabbitMQ Cluster

```hcl
# cluster.tf
resource "kubernetes_manifest" "rabbitmq_cluster" {
  depends_on = [helm_release.rabbitmq_operator]

  manifest = {
    apiVersion = "rabbitmq.com/v1beta1"
    kind       = "RabbitmqCluster"
    metadata = {
      name      = "production-rabbitmq"
      namespace = kubernetes_namespace.rabbitmq.metadata[0].name
    }
    spec = {
      replicas = 3  # 3-node cluster for HA

      resources = {
        requests = {
          cpu    = "500m"
          memory = "1Gi"
        }
        limits = {
          cpu    = "2"
          memory = "2Gi"
        }
      }

      persistence = {
        storageClassName = var.storage_class
        storage          = "20Gi"
      }

      rabbitmq = {
        additionalConfig = <<-CONFIG
          # Enable management plugin
          management.tcp.port = 15672
          # Set disk free limit
          disk_free_limit.relative = 1.5
          # Cluster high-availability settings
          cluster_partition_handling = pause_minority
        CONFIG

        additionalPlugins = [
          "rabbitmq_management",
          "rabbitmq_shovel",
          "rabbitmq_shovel_management",
          "rabbitmq_prometheus",
        ]
      }

      service = {
        type = "ClusterIP"
      }

      tls = {
        # Enable TLS for AMQP connections
        secretName          = "rabbitmq-tls"
        caSecretName        = "rabbitmq-ca"
        disableNonTLSListeners = true
      }
    }
  }
}
```

## Declaring Queues and Exchanges

```hcl
# topology.tf
# Create a queue via the Topology Operator
resource "kubernetes_manifest" "orders_queue" {
  depends_on = [kubernetes_manifest.rabbitmq_cluster]

  manifest = {
    apiVersion = "rabbitmq.com/v1beta1"
    kind       = "Queue"
    metadata = {
      name      = "orders-queue"
      namespace = kubernetes_namespace.rabbitmq.metadata[0].name
    }
    spec = {
      name    = "orders"
      durable = true

      arguments = {
        "x-message-ttl"          = 86400000  # 24 hours TTL
        "x-dead-letter-exchange" = "orders-dlx"
        "x-max-length"           = 100000
      }

      rabbitmqClusterReference = {
        name = kubernetes_manifest.rabbitmq_cluster.manifest.metadata.name
      }
    }
  }
}

# Create an exchange for fanout routing
resource "kubernetes_manifest" "orders_exchange" {
  depends_on = [kubernetes_manifest.rabbitmq_cluster]

  manifest = {
    apiVersion = "rabbitmq.com/v1beta1"
    kind       = "Exchange"
    metadata = {
      name      = "orders-exchange"
      namespace = kubernetes_namespace.rabbitmq.metadata[0].name
    }
    spec = {
      name    = "orders"
      type    = "topic"
      durable = true

      rabbitmqClusterReference = {
        name = kubernetes_manifest.rabbitmq_cluster.manifest.metadata.name
      }
    }
  }
}
```

## Best Practices

- Deploy with 3 replicas (odd number) to maintain quorum in split-brain scenarios.
- Enable the Prometheus plugin and scrape metrics for monitoring queue depth, consumer counts, and message rates.
- Use quorum queues instead of classic mirrored queues — they are more reliable and have better performance.
- Set `x-dead-letter-exchange` on queues so rejected/expired messages are routed to a DLQ rather than silently dropped.
- Use TLS for all AMQP connections in production — the default unencrypted connections are not appropriate for sensitive data.
