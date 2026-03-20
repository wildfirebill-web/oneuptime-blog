# How to Set Up Event-Driven Architecture on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Event-Driven, Kafka, NATS, Kubernetes, Microservices

Description: Guide to building event-driven architectures on Rancher using Kafka, NATS, and Knative Eventing for decoupled microservices.

## Introduction

Event-driven architecture (EDA) enables loosely coupled, highly scalable systems where components communicate through events rather than direct API calls. Rancher provides the platform to run the messaging infrastructure and event consumers at scale.

## EDA Components on Rancher

- **Event Broker**: Kafka, NATS, RabbitMQ
- **Event Sources**: Applications producing events
- **Event Consumers**: Functions and services processing events
- **Event Router**: Knative Eventing, KEDA

## Step 1: Deploy Apache Kafka

```bash
# Install Strimzi Kafka Operator

kubectl create namespace kafka
kubectl apply -f https://strimzi.io/install/latest?namespace=kafka -n kafka

# Wait for operator
kubectl wait pods --all \
  --for=condition=Ready \
  --namespace kafka \
  --timeout=300s
```

```yaml
# kafka-cluster.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: production-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3               # 3 brokers for HA
    listeners:
    - name: plain
      port: 9092
      type: internal
      tls: false
    - name: tls
      port: 9093
      type: internal
      tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        class: longhorn
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: longhorn
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

## Step 2: Create Kafka Topics

```yaml
# kafka-topics.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: user-events
  namespace: kafka
  labels:
    strimzi.io/cluster: production-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000    # 7 days
    segment.bytes: 1073741824  # 1GB segments
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-events
  namespace: kafka
  labels:
    strimzi.io/cluster: production-kafka
spec:
  partitions: 24               # More partitions for higher throughput
  replicas: 3
```

## Step 3: Set Up NATS as Lightweight Broker

```bash
# Install NATS with JetStream
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update

helm install nats nats/nats \
  --namespace messaging \
  --create-namespace \
  --set config.cluster.enabled=true \
  --set config.cluster.replicas=3 \
  --set config.jetstream.enabled=true \
  --set config.jetstream.fileStore.pvc.size=50Gi
```

## Step 4: Configure Knative Eventing Brokers

```yaml
# knative-broker.yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: production
  annotations:
    eventing.knative.dev/broker.class: MTChannelBasedBroker
spec:
  config:
    apiVersion: v1
    kind: ConfigMap
    name: kafka-channel-config
    namespace: knative-eventing
  delivery:
    backoffDelay: PT2S         # Retry after 2 seconds
    backoffPolicy: exponential
    retry: 5                   # Retry 5 times
    deadLetterSink:
      ref:
        apiVersion: v1
        kind: Service
        name: dead-letter-handler
```

## Step 5: Create Event Sources

```yaml
# kafka-event-source.yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: order-source
  namespace: production
spec:
  consumerGroup: knative-order-consumer
  bootstrapServers:
  - production-kafka-kafka-bootstrap.kafka:9092
  topics:
  - order-events
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
```

## Step 6: Create Event Triggers

```yaml
# order-trigger.yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: process-orders
  namespace: production
spec:
  broker: default
  filter:
    attributes:
      type: com.example.order.created  # Filter by event type
      source: order-service
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: send-notifications
  namespace: production
spec:
  broker: default
  filter:
    attributes:
      type: com.example.order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: notification-service
```

## Step 7: Event Schema Registry

```bash
# Install Confluent Schema Registry
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts
helm install schema-registry confluentinc/cp-schema-registry \
  --namespace kafka \
  --set kafka.bootstrapServers=production-kafka-kafka-bootstrap.kafka:9092

# Register an event schema
curl -X POST http://schema-registry.kafka:8081/subjects/order-events-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"OrderEvent\",\"fields\":[{\"name\":\"orderId\",\"type\":\"string\"},{\"name\":\"customerId\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"}]}"
  }'
```

## Conclusion

Event-driven architecture on Rancher enables building resilient, decoupled systems that scale independently. By combining Kafka or NATS as the event backbone with Knative Eventing for routing and serverless functions for processing, you create a powerful EDA platform. Start with a simple event producer-consumer pattern and evolve towards complex event choreography as your system grows.
