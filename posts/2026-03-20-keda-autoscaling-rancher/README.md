# How to Configure KEDA Autoscaling in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, keda, autoscaling, kubernetes, event-driven, scaling

Description: Complete guide to installing and configuring KEDA (Kubernetes Event-Driven Autoscaling) in Rancher for event-based workload scaling.

## Introduction

KEDA (Kubernetes Event-Driven Autoscaling) extends the Kubernetes Horizontal Pod Autoscaler to support scaling based on external event sources like Kafka, RabbitMQ, Prometheus metrics, and more. It enables scale-to-zero for event-driven workloads.

## KEDA Architecture

KEDA adds three components to Kubernetes:
1. **KEDA Operator**: Manages ScaledObjects and ScaledJobs
2. **Metrics Server**: Exposes external metrics to Kubernetes HPA
3. **Scalers**: Adapters for different event sources (50+ built-in)

## Step 1: Install KEDA

```bash
# Add KEDA Helm repository
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install KEDA
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.12.0 \
  --set operator.replicaCount=2 \
  --set metricsServer.replicaCount=2 \
  --set operator.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].key=app \
  --set operator.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].operator=In \
  --set operator.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].labelSelector.matchExpressions[0].values[0]=keda-operator \
  --set operator.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].topologyKey=kubernetes.io/hostname

# Verify installation
kubectl get pods -n keda
kubectl get crds | grep keda
```

## Step 2: Scale with Kafka Trigger

```yaml
# kafka-consumer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-consumer
  template:
    metadata:
      labels:
        app: kafka-consumer
    spec:
      containers:
      - name: consumer
        image: registry.example.com/kafka-consumer:latest
        env:
        - name: KAFKA_BROKERS
          value: "kafka.kafka.svc.cluster.local:9092"
        - name: KAFKA_TOPIC
          value: "events-input"
---
# kafka-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  
  pollingInterval: 15          # Check metrics every 15 seconds
  cooldownPeriod: 300          # Wait 5 min before scaling down
  idleReplicaCount: 0          # Scale to zero when idle
  minReplicaCount: 0
  maxReplicaCount: 50
  
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.kafka.svc.cluster.local:9092
      consumerGroup: keda-consumer-group
      topic: events-input
      lagThreshold: "100"       # Scale up when lag > 100 per replica
      activationLagThreshold: "5"  # Activate (from 0) when lag > 5
      offsetResetPolicy: latest
```

## Step 3: Scale with Prometheus Metrics

```yaml
# prometheus-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-deployment
  minReplicaCount: 2
  maxReplicaCount: 50
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.cattle-monitoring-system.svc.cluster.local:9090
      metricName: http_requests_per_second
      threshold: "100"          # 100 RPS per replica
      activationThreshold: "10" # Start scaling at 10 RPS
      query: |
        sum(rate(http_requests_total{
          job="api-deployment"
        }[2m]))
```

## Step 4: Scale Jobs Based on Queue Length

```yaml
# rabbitmq-scaledjob.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor-job
  namespace: default
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: image-processor
          image: registry.example.com/image-processor:latest
          env:
          - name: RABBITMQ_URL
            valueFrom:
              secretKeyRef:
                name: rabbitmq-credentials
                key: url
        restartPolicy: Never
  
  # Job scaling configuration
  pollingInterval: 10
  maxReplicaCount: 30
  rollout:
    strategy: default
    propagationPolicy: foreground
  
  triggers:
  - type: rabbitmq
    authenticationRef:
      name: rabbitmq-trigger-auth
    metadata:
      protocol: amqp
      queueName: image-processing-queue
      mode: QueueLength
      value: "5"                # 1 job per 5 messages
      activationValue: "1"
```

## Step 5: Trigger Authentication

```yaml
# keda-auth.yaml - Manage credentials securely
apiVersion: v1
kind: Secret
metadata:
  name: kafka-credentials
  namespace: default
data:
  username: a2Fma2EtdXNlcg==     # base64 encoded
  password: c2VjcmV0cGFzcw==     # base64 encoded
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-trigger-auth
  namespace: default
spec:
  secretTargetRef:
  - parameter: username
    name: kafka-credentials
    key: username
  - parameter: password
    name: kafka-credentials
    key: password
```

## Step 6: Scale on Cron Schedule

```yaml
# cron-scaledobject.yaml - Business hours scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: business-hours-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: web-app
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 8 * * 1-5"      # Monday-Friday 8 AM
      end: "0 20 * * 1-5"       # Monday-Friday 8 PM
      desiredReplicas: "10"      # Scale up during business hours
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 20 * * 1-5"     # After hours
      end: "0 8 * * 1-5"
      desiredReplicas: "2"       # Scale down after hours
```

## Monitoring KEDA

```bash
# Check all scaled objects
kubectl get scaledobject -A

# Check KEDA metrics
kubectl get hpa -A | grep keda

# View KEDA operator logs
kubectl logs -n keda \
  -l app=keda-operator \
  --tail=50

# Check scaler status
kubectl describe scaledobject kafka-consumer-scaler -n default
```

## Conclusion

KEDA transforms Rancher clusters into event-driven computing platforms where workloads automatically scale based on actual demand signals from Kafka, RabbitMQ, Prometheus, and 50+ other sources. Scale-to-zero reduces costs for variable workloads while instant scale-up ensures responsiveness. KEDA works alongside standard Kubernetes HPA for a complete autoscaling solution.
