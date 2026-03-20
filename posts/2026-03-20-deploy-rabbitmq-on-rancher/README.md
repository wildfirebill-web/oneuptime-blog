# How to Deploy RabbitMQ on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, RabbitMQ, Kubernetes, Message Queues, Helm, AMQP

Description: Deploy a production-ready RabbitMQ cluster on Rancher using Helm with persistent storage, management UI access, and proper resource configuration.

## Introduction

RabbitMQ is a widely used open-source message broker supporting AMQP, MQTT, and STOMP protocols. Running it on Rancher enables automatic pod recovery, horizontal scaling, and seamless integration with cloud-native applications.

## Prerequisites

- Rancher cluster with at least 3 nodes for HA deployment
- `helm` and `kubectl` available
- A StorageClass for persistent volumes

## Step 1: Add Bitnami Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## Step 2: Configure Values

```yaml
# rabbitmq-values.yaml

auth:
  username: admin
  password: "securepassword"
  erlangCookie: "your-erlang-cookie-secret"   # Must be consistent across replicas

replicaCount: 3   # Three-node cluster for HA

persistence:
  enabled: true
  storageClass: "longhorn"
  size: 20Gi

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1"

metrics:
  enabled: true   # Enable Prometheus metrics endpoint
  serviceMonitor:
    enabled: false   # Set true if Prometheus Operator is installed

plugins: "rabbitmq_management rabbitmq_peer_discovery_k8s rabbitmq_prometheus"
```

## Step 3: Deploy RabbitMQ

```bash
kubectl create namespace messaging

helm install rabbitmq bitnami/rabbitmq \
  --namespace messaging \
  --values rabbitmq-values.yaml
```

## Step 4: Verify the Cluster

```bash
# Check all pods are running
kubectl get pods -n messaging

# Check cluster status from inside a pod
kubectl exec -it rabbitmq-0 -n messaging -- rabbitmqctl cluster_status
```

A healthy cluster shows all three nodes listed under `running_nodes`.

## Step 5: Access the Management UI

```bash
# Port-forward to access the RabbitMQ management console
kubectl port-forward svc/rabbitmq -n messaging 15672:15672

# Open http://localhost:15672 and log in with admin/securepassword
```

## Step 6: Create a Test Queue

Use the management CLI to create a queue and verify publishing.

```bash
# Exec into a pod to use rabbitmqadmin
kubectl exec -it rabbitmq-0 -n messaging -- bash

# Declare a test queue
rabbitmqadmin declare queue name=test-queue durable=true

# Publish a test message
rabbitmqadmin publish exchange=amq.default routing_key=test-queue payload="Hello, RabbitMQ!"

# Get the message
rabbitmqadmin get queue=test-queue
```

## Step 7: Configure a Service for Applications

Expose RabbitMQ to other pods in the cluster via the headless service created by the Helm chart.

```yaml
# Application environment variable pointing to RabbitMQ
env:
  - name: RABBITMQ_URL
    value: "amqp://admin:securepassword@rabbitmq.messaging.svc.cluster.local:5672"
```

## Conclusion

Your RabbitMQ cluster is now running on Rancher with HA enabled through a three-node StatefulSet. The `rabbitmq_peer_discovery_k8s` plugin handles node discovery automatically using Kubernetes service endpoints. Monitor queue depths and message rates through the management UI or Prometheus integration.
