# How to Scale Message Queue Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Scaling, Message Queue, Kafka, RabbitMQ, Kubernetes HPA

Description: Learn how to horizontally scale message queue clusters in Rancher by adding brokers, rebalancing partitions, and using KEDA for consumer autoscaling.

## Introduction

Message queue scaling involves two distinct concerns: scaling the broker cluster itself (adding nodes to increase throughput) and scaling consumers (increasing consumer replicas to process more messages). This guide covers both approaches in Rancher.

## Scaling Kafka Brokers

Kafka brokers are deployed as StatefulSets. Scaling up requires updating the replica count and rebalancing partitions.

```bash
# Scale Kafka from 3 to 5 brokers
kubectl scale statefulset kafka-controller \
  --replicas=5 \
  -n messaging

# Verify new pods are scheduled
kubectl get pods -n messaging -l app.kubernetes.io/name=kafka
```

After adding brokers, existing partitions must be reassigned to use the new brokers:

```bash
# Inside a Kafka pod, generate a reassignment plan
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2,3,4" \
  --generate

# Execute the reassignment
kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute
```

## Scaling RabbitMQ Clusters

RabbitMQ uses quorum queues for HA. Scale by updating the StatefulSet replica count:

```bash
# Scale RabbitMQ from 3 to 5 nodes
kubectl scale statefulset rabbitmq \
  --replicas=5 \
  -n messaging

# Verify cluster membership
kubectl exec -it rabbitmq-0 -n messaging -- \
  rabbitmqctl cluster_status
```

## Autoscaling Consumers with KEDA

KEDA (Kubernetes Event-Driven Autoscaling) scales consumer deployments based on queue depth.

```yaml
# keda-kafka-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: order-processor    # Deployment to scale
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.messaging.svc.cluster.local:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "100"     # Scale up when lag exceeds 100 messages
        offsetResetPolicy: latest
```

```bash
kubectl apply -f keda-kafka-scaler.yaml
```

## Autoscaling RabbitMQ Consumers with KEDA

```yaml
# keda-rabbitmq-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: email-worker
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: email-queue
        mode: QueueLength
        value: "50"    # Scale when queue has more than 50 messages
      authenticationRef:
        name: rabbitmq-auth
```

## Conclusion

Scaling message queue clusters in Rancher involves two layers: broker cluster scaling via StatefulSet replica adjustments and consumer autoscaling via KEDA. KEDA's event-driven scaling ensures consumer capacity always matches queue depth, preventing message backlogs while avoiding wasted resources during idle periods.
