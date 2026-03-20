# How to Deploy Apache Kafka on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Kafka, Message Queues, Streaming, Strimzi

Description: Deploy Apache Kafka on Rancher-managed Kubernetes clusters using the Strimzi Kafka Operator for distributed event streaming and message processing.

## Introduction

Apache Kafka is a distributed event streaming platform used by thousands of companies for high-throughput, fault-tolerant message processing. The Strimzi operator provides Kubernetes-native management of Kafka clusters, simplifying deployment, configuration, and operations. This guide covers deploying a production Kafka cluster on Rancher using Strimzi.

## Prerequisites

- Rancher-managed Kubernetes cluster with at least 3 nodes
- Helm 3.x installed
- kubectl with cluster-admin access
- A StorageClass for persistent volumes

## Step 1: Install Strimzi Operator

```bash
# Install Strimzi Kafka Operator via Helm

helm repo add strimzi https://strimzi.io/charts/
helm repo update

helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --namespace kafka \
  --create-namespace \
  --set watchNamespaces="{kafka,messaging}" \
  --wait

# Verify the operator is running
kubectl get pods -n kafka -l name=strimzi-cluster-operator
```

## Step 2: Deploy Kafka Cluster

```yaml
# kafka-cluster.yaml - Production Kafka cluster with Strimzi
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-prod
  namespace: kafka
spec:
  kafka:
    version: 3.7.0
    replicas: 3

    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      # External access via Load Balancer
      - name: external
        port: 9094
        type: loadbalancer
        tls: true

    config:
      # Replication factor for internal topics
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      # Log retention
      log.retention.hours: 168  # 7 days
      log.retention.bytes: 10737418240  # 10GB per partition
      # Compression
      compression.type: snappy
      # Network tuning
      num.network.threads: 5
      num.io.threads: 8
      socket.send.buffer.bytes: 102400
      socket.receive.buffer.bytes: 102400

    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 100Gi
          class: standard
          deleteClaim: false

    resources:
      requests:
        memory: 4Gi
        cpu: "1"
      limits:
        memory: 8Gi
        cpu: "4"

    jvmOptions:
      -Xms: 2048m
      -Xmx: 4096m
      -XX:
        UseG1GC: true
        MaxGCPauseMillis: 20
        InitiatingHeapOccupancyPercent: 35

    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml

  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: standard
      deleteClaim: false
    resources:
      requests:
        memory: 1Gi
        cpu: 500m

  entityOperator:
    topicOperator:
      resources:
        requests:
          memory: 256Mi
          cpu: 100m
    userOperator:
      resources:
        requests:
          memory: 256Mi
          cpu: 100m

  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
```

## Step 3: Create Kafka Topics

```yaml
# kafka-topics.yaml - Declarative topic management
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-prod
spec:
  partitions: 12      # Number of partitions for parallelism
  replicas: 3         # Replication factor
  config:
    retention.ms: 604800000     # 7 days
    segment.bytes: 1073741824   # 1GB segments
    compression.type: snappy
    cleanup.policy: delete
    min.insync.replicas: "2"    # Ensure at least 2 replicas in sync
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders-dlq
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-prod
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 2592000000  # 30 days for DLQ
    cleanup.policy: compact   # Keep latest message per key
```

## Step 4: Create Kafka Users with ACLs

```yaml
# kafka-user.yaml - Kafka user with specific ACLs
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: order-producer
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-prod
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
      # Allow writing to orders topic
      - resource:
          type: topic
          name: orders-topic
          patternType: literal
        operations:
          - Write
          - Describe
        host: "*"
      # Allow transactional writes
      - resource:
          type: transactionalId
          name: order-producer-txn
          patternType: literal
        operations:
          - Write
          - Describe
        host: "*"
```

## Step 5: Connect Applications to Kafka

```yaml
# app-deployment.yaml - Application using Kafka
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:v1.0
          env:
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "kafka-prod-kafka-bootstrap.kafka.svc.cluster.local:9092"
            - name: KAFKA_TOPIC
              value: "orders-topic"
            - name: KAFKA_GROUP_ID
              value: "order-processor-group"
            # TLS configuration
            - name: KAFKA_SECURITY_PROTOCOL
              value: "SSL"
            - name: KAFKA_SSL_TRUSTSTORE_LOCATION
              value: "/opt/kafka/certs/truststore.jks"
            - name: KAFKA_SSL_KEYSTORE_LOCATION
              value: "/opt/kafka/certs/keystore.jks"
          volumeMounts:
            - name: kafka-certs
              mountPath: /opt/kafka/certs
      volumes:
        - name: kafka-certs
          secret:
            secretName: order-producer  # KafkaUser secret
```

## Step 6: Deploy Kafka Connect

```yaml
# kafka-connect.yaml - Kafka Connect for data integration
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: kafka-connect
  namespace: kafka
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 3.7.0
  replicas: 2
  bootstrapServers: kafka-prod-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: kafka-prod-cluster-ca-cert
        certificate: ca.crt
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    config.storage.replication.factor: 3
    offset.storage.replication.factor: 3
    status.storage.replication.factor: 3
```

## Step 7: Monitor Kafka Cluster

```yaml
# kafka-metrics-configmap.yaml - JMX Prometheus metrics config
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-metrics
  namespace: kafka
data:
  kafka-metrics-config.yml: |
    lowercaseOutputName: true
    rules:
      - pattern: "kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value"
        name: kafka_server_$1_$2
        type: GAUGE
      - pattern: "kafka.network<type=(.+), name=(.+)><>Count"
        name: kafka_network_$1_$2_count
        type: COUNTER
```

## Troubleshooting

```bash
# Check cluster status
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-broker-api-versions.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092

# List topics
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-topics.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092 \
  --list

# Check consumer group lag
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092 \
  --describe --group order-processor-group

# Check logs
kubectl logs -n kafka kafka-prod-kafka-0 --tail=100
```

## Conclusion

Apache Kafka on Rancher with the Strimzi operator provides a production-grade event streaming platform with Kubernetes-native management. Strimzi handles complex operational tasks like rolling upgrades, TLS certificate management, and user authorization. For production deployments, use TLS authentication, configure appropriate replication factors, and monitor consumer lag to detect processing bottlenecks.
