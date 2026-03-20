# How to Configure Serverless Autoscaling in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Serverless, Autoscaling, Kubernetes, KEDA, Knative

Description: Guide to configuring autoscaling for serverless workloads in Rancher using KEDA, Knative, and HPA for demand-driven scaling.

## Introduction

Serverless autoscaling automatically adjusts the number of running function instances based on demand, scaling to zero when idle and scaling up to handle traffic bursts. This guide covers configuring autoscaling in Rancher serverless environments.

## Autoscaling Strategies

1. **Request-based scaling**: Scale based on concurrent requests
2. **Queue-based scaling**: Scale based on queue depth
3. **CPU/Memory scaling**: Traditional HPA
4. **Custom metrics scaling**: KEDA for any metric source

## Knative Autoscaling Configuration

```yaml
# knative-autoscaling.yaml

apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: auto-scaling-demo
  namespace: default
spec:
  template:
    metadata:
      annotations:
        # KPA (Knative Pod Autoscaler) settings
        autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
        
        # Scale to zero
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "20"
        
        # Target concurrent requests per pod
        autoscaling.knative.dev/target: "50"
        
        # Utilization percentage of target before scaling
        autoscaling.knative.dev/target-utilization-percentage: "70"
        
        # Scale-up/down rates
        autoscaling.knative.dev/scale-up-rate: "10"
        autoscaling.knative.dev/scale-down-rate: "1.0"
        
        # Panic mode: aggressive scale-up when 200% of target
        autoscaling.knative.dev/panic-threshold-percentage: "200"
        autoscaling.knative.dev/panic-window-percentage: "10"
    spec:
      containers:
      - image: registry.example.com/my-function:latest
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
```

## KEDA Installation

```bash
# Install KEDA via Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.12.0 \
  --set operator.replicaCount=2
```

## KEDA ScaledObject for HTTP Workloads

```yaml
# keda-http-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: http-function-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: my-function-deployment
  
  # Scaling bounds
  minReplicaCount: 0           # Scale to zero
  maxReplicaCount: 30
  
  # How quickly to scale down to zero
  cooldownPeriod: 60           # 60 seconds after last request
  
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.cattle-monitoring-system.svc.cluster.local:9090
      metricName: http_requests_total
      threshold: "10"           # 10 RPS per replica
      query: |
        sum(rate(nginx_ingress_controller_requests{
          service="my-function-service"
        }[1m]))
```

## KEDA with Kafka Trigger

```yaml
# keda-kafka-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: kafka-consumer-deployment
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.default.svc.cluster.local:9092
      consumerGroup: my-consumer-group
      topic: events-input
      lagThreshold: "100"       # 100 messages per replica
      activationLagThreshold: "5"
```

## OpenFaaS Autoscaling

```yaml
# openfaas-function with autoscaling labels
functions:
  data-processor:
    lang: python3
    handler: ./data-processor
    image: registry.example.com/functions/data-processor:latest
    labels:
      # OpenFaaS autoscaling labels
      com.openfaas.scale.min: "1"
      com.openfaas.scale.max: "20"
      com.openfaas.scale.factor: "25"      # Scale by 25% each time
      com.openfaas.scale.type: "rps"       # Scale on RPS
      com.openfaas.scale.target: "50"      # Target 50 RPS per replica
      com.openfaas.scale.zero: "true"
      com.openfaas.scale.zero-duration: "2m"
```

## Cluster Autoscaler Integration

For serverless workloads that need node-level scaling:

```yaml
# cluster-autoscaler.yaml - Rancher provisioned cluster
apiVersion: management.cattle.io/v3
kind: Cluster
metadata:
  name: my-cluster
spec:
  rkeConfig:
    machinePools:
    - name: worker-pool
      quantity: 2
      machineConfigRef:
        kind: AWSNodeTemplate
        name: worker-template
      # Cluster autoscaler annotations
      annotations:
        cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "2"
        cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "20"
```

## Monitoring Autoscaling

```bash
# Watch Knative pod autoscaler
kubectl get kpa -A -w

# Watch KEDA scaled objects
kubectl get scaledobject -A

# Watch HPA
kubectl get hpa -A -w

# Prometheus query for scaling events
# knative_serving_autoscaler_actual_pods
# keda_scaler_active
```

## Conclusion

Serverless autoscaling in Rancher can be achieved through multiple mechanisms: Knative's built-in KPA for HTTP workloads, KEDA for event-driven scaling based on external metrics, and OpenFaaS's built-in autoscaler. Choose the approach that matches your trigger source and scaling requirements.
