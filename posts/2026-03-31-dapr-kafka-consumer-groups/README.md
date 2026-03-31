# How to Tune Kafka Consumer Groups for Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kafka, Consumer Group, Pub/Sub, Performance, Tuning, Microservice

Description: Tune Kafka consumer group settings in Dapr pub/sub for optimal throughput, partition assignment, and lag management in production deployments.

---

## Overview

Kafka consumer groups allow multiple instances of a service to consume messages in parallel. When using Dapr with Kafka pub/sub, understanding how Dapr maps consumer groups to Kafka concepts is critical for achieving desired throughput and avoiding rebalancing issues. This guide covers key tuning parameters and strategies.

## How Dapr Uses Kafka Consumer Groups

Dapr uses the `consumerGroup` metadata field as the Kafka consumer group ID. All instances of the same Dapr application (same `app-id`) share one consumer group. Kafka assigns partitions to active consumers within the group, so scaling your service adds parallel consumers.

## Dapr Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka-0.kafka.default.svc.cluster.local:9092"
    - name: consumerGroup
      value: "order-processor"
    - name: authType
      value: "none"
    - name: maxMessageBytes
      value: "1048576"
    - name: fetchMessageMaxBytes
      value: "1048576"
    - name: consumeRetryInterval
      value: "200ms"
    - name: initialOffset
      value: "newest"
```

## Key Tuning Parameters

### Session Timeout and Heartbeat

Configure via Kafka broker and component settings to prevent unnecessary rebalancing:

```yaml
metadata:
  - name: sessionTimeout
    value: "10000"
  - name: heartbeatInterval
    value: "3000"
```

The heartbeat interval should be one-third of the session timeout. If your processing takes longer than the session timeout, Kafka will trigger a rebalance.

### Max Poll Interval

For slow processing tasks, increase the max poll interval:

```yaml
metadata:
  - name: maxPollIntervalMs
    value: "300000"
```

## Scaling Consumers

To scale consumption, increase the number of Dapr sidecar replicas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: app
          image: order-processor:latest
        - name: dapr-sidecar
          # Dapr injects automatically via annotation
```

Add Dapr annotations:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-processor"
  dapr.io/app-port: "5000"
```

Kafka will distribute partitions across up to `N` replicas, where `N` equals the partition count on the topic.

## Monitoring Consumer Lag

```bash
# Check consumer group lag
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group order-processor

# Output shows CURRENT-OFFSET, LOG-END-OFFSET, and LAG per partition
```

## Rebalancing Strategies

Use `CooperativeStickyAssignor` to reduce rebalancing disruption:

```yaml
metadata:
  - name: groupInstanceID
    value: "order-processor-${HOSTNAME}"
```

Setting a `groupInstanceID` enables static membership, preventing rebalances when pods restart within the session timeout window.

## Processing Failures

When Dapr receives a non-200 response from your app, it retries based on `consumeRetryInterval`. Tune this to balance retry speed against broker load:

```yaml
metadata:
  - name: consumeRetryInterval
    value: "500ms"
  - name: consumeRetryMaxElapsedTime
    value: "15s"
```

## Summary

Tuning Kafka consumer groups in Dapr involves balancing session timeouts, heartbeat intervals, and partition counts against your deployment scale. Enable static membership with `groupInstanceID` to reduce rebalancing during rolling restarts. Monitor consumer lag to detect processing bottlenecks and scale replicas up to the partition count for maximum parallelism.
