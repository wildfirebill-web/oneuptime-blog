# How to Set Up Namespace Consumer Groups in Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Consumer Group, Namespace, Kubernetes

Description: Learn how to configure namespace-scoped consumer groups in Dapr pub/sub to isolate message consumption between teams or environments sharing a broker.

---

## What Are Namespace Consumer Groups?

When multiple Dapr applications subscribe to the same topic on a shared message broker (like Kafka or Azure Service Bus), you often want each application to receive its own copy of every message - not compete for messages. Consumer groups allow this. Dapr supports namespace-level consumer group scoping so that apps in different Kubernetes namespaces get independent consumer groups automatically.

## How Dapr Names Consumer Groups

By default, Dapr generates the consumer group name using the app ID. When `consumerID` is set to `{namespace}` or uses the app ID with namespace metadata, each namespace gets its own group.

For Kafka-backed pub/sub, configure the consumer ID explicitly:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: team-a
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: kafka-broker:9092
  - name: consumerID
    value: team-a-consumer-group
  - name: authType
    value: none
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: team-b
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: kafka-broker:9092
  - name: consumerID
    value: team-b-consumer-group
  - name: authType
    value: none
```

Each namespace has its own component definition with a unique `consumerID`, so `team-a` and `team-b` both receive every message from the shared topic independently.

## Dynamic Consumer ID Using App ID

For components that support it, you can use the Dapr app ID as the consumer group to ensure uniqueness without hardcoding:

```yaml
metadata:
- name: consumerID
  value: "{appID}"
```

This substitutes the Dapr application ID at runtime, giving each app its own consumer group.

## Namespace Consumer Groups with Azure Service Bus

For Azure Service Bus, Dapr uses subscription names that you can scope per namespace:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: production
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: asb-secret
      key: connectionString
  - name: consumerID
    value: production-order-service
```

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: staging
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: asb-secret
      key: connectionString
  - name: consumerID
    value: staging-order-service
```

Both `production` and `staging` subscribe to the same topic but maintain separate cursor positions.

## Verifying Consumer Group Isolation

With Kafka, check that separate consumer groups exist:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-broker:9092 \
  --list
# Expected output:
# team-a-consumer-group
# team-b-consumer-group
```

Check each group's offsets independently:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-broker:9092 \
  --group team-a-consumer-group \
  --describe
```

## Publishing a Test Message

```bash
curl -s -X POST http://localhost:3500/v1.0/publish/pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "ord-001", "source": "web"}'
```

Both `team-a` and `team-b` services will receive this message independently via their separate consumer groups.

## Summary

Dapr namespace consumer groups enable independent message consumption across teams or environments sharing a single broker by configuring unique `consumerID` values per namespace. This is especially important with Kafka and Azure Service Bus, where consumer group offset tracking is per-group, ensuring that each namespace's services receive all messages without competing with or interfering with others.
