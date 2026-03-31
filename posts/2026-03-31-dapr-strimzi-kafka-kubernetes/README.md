# How to Use Dapr with Strimzi Kafka on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kafka, Strimzi, Kubernetes, Pub/Sub

Description: Step-by-step guide to connecting Dapr's pub/sub building block with a Strimzi-managed Kafka cluster running on Kubernetes.

---

## Overview

Strimzi is a Kubernetes operator that simplifies deploying and managing Apache Kafka on Kubernetes. Paired with Dapr, you get a cloud-native pub/sub backbone with operator-managed lifecycle and Dapr's portable API.

## Deploying Strimzi Kafka

Install the Strimzi operator and create a Kafka cluster:

```bash
kubectl create namespace kafka
kubectl apply -f https://strimzi.io/install/latest?namespace=kafka -n kafka
```

Create a basic Kafka cluster:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
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
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 10Gi
          deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 5Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```bash
kubectl apply -f kafka-cluster.yaml -n kafka
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```

## Creating a Kafka Topic

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: dapr-events
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 6
  replicas: 3
  config:
    retention.ms: 604800000
    segment.bytes: 1073741824
```

## Configuring Dapr to Use Strimzi Kafka

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: strimzi-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092"
    - name: authType
      value: "none"
    - name: consumerGroup
      value: "dapr-consumer-group"
    - name: initialOffset
      value: "newest"
```

## Publishing and Subscribing

Deploy a publisher service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: publisher
spec:
  replicas: 1
  selector:
    matchLabels:
      app: publisher
  template:
    metadata:
      labels:
        app: publisher
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "publisher"
        dapr.io/app-port: "8080"
    spec:
      containers:
        - name: publisher
          image: myregistry/publisher:latest
          ports:
            - containerPort: 8080
```

Publish a message from within the service:

```bash
curl -X POST http://localhost:3500/v1.0/publish/strimzi-pubsub/dapr-events \
  -H "Content-Type: application/json" \
  -d '{"eventType": "user.signup", "userId": "u-456"}'
```

Subscribe with a declarative subscription:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: events-subscription
spec:
  pubsubname: strimzi-pubsub
  topic: dapr-events
  route: /events
  scopes:
    - subscriber-app
```

## Monitoring with Strimzi Kafka Exporter

```bash
kubectl apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/main/examples/metrics/kafka-metrics.yaml -n kafka
```

This exposes consumer lag and partition metrics to Prometheus for observability.

## Summary

Strimzi provides a production-grade Kafka deployment on Kubernetes through a CRD-based operator model. Dapr's pub/sub component connects to Strimzi Kafka with a simple broker address, providing a portable API for your microservices. Together they deliver a fully Kubernetes-native event streaming platform with operator-managed lifecycle management.
