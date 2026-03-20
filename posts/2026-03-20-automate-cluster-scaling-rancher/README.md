# How to Automate Cluster Scaling in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cluster Scaling, Autoscaler, Kubernetes, HPA, VPA, KEDA

Description: Automate cluster scaling in Rancher using Cluster Autoscaler for node-level scaling, HPA/VPA for pod-level scaling, and KEDA for event-driven scaling to handle variable workloads efficiently.

## Introduction

Cluster scaling automation in Rancher operates at three levels: pod scaling (more replicas of existing pods), vertical pod scaling (more CPU/memory per pod), and node scaling (adding/removing nodes from the cluster). Each level requires different tools and configurations. Automating all three creates a fully elastic cluster that scales with demand and shrinks during off-peak hours.

## Level 1: Horizontal Pod Autoscaler (HPA)

```yaml
# HPA scales pod count based on CPU/memory metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

## Level 2: Vertical Pod Autoscaler (VPA)

```yaml
# VPA adjusts CPU/memory requests based on actual usage history
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Auto"    # Recreate pods with new resource requests
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "4"
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
```

## Level 3: Cluster Autoscaler

```yaml
# Cluster Autoscaler on AWS (EC2 node groups)
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=production-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/ClusterAutoscalerRole

# Node group configuration
# Tag EC2 autoscaling groups:
# k8s.io/cluster-autoscaler/enabled = true
# k8s.io/cluster-autoscaler/production-cluster = owned
# k8s.io/cluster-autoscaler/node-template/resources/ephemeral-storage = 100Gi
```

For bare-metal with Rancher's Machine Provisioning:

```yaml
# RKE2 cluster with machine pools and auto-scaling
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production
spec:
  rkeConfig:
    machinePools:
      - name: workers
        quantity: 3           # Base quantity
        machineConfigRef:
          kind: VmwarevsphereConfig
          name: worker-vm
        roles: [worker]
        # Rancher machine pool auto-scaling
        minSize: 3
        maxSize: 10
```

## Level 4: KEDA Event-Driven Scaling

```yaml
# Scale workers based on Kafka queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: kafka-consumer
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.kafka.svc:9092
        consumerGroup: consumer-group-1
        topic: order-events
        lagThreshold: "50"    # Scale when lag exceeds 50 messages/consumer

---
# Scale based on RabbitMQ queue length
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaler
spec:
  scaleTargetRef:
    name: message-processor
  triggers:
    - type: rabbitmq
      metadata:
        queueName: processing-queue
        queueLength: "10"    # 1 pod per 10 messages
        hostFromEnv: RABBITMQ_URI
```

## Scheduled Scaling

```yaml
# Scale down at night to save costs
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: business-hours-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: web-frontend
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: cron
      metadata:
        timezone: "America/New_York"
        start: "0 8 * * 1-5"    # Scale up at 8 AM weekdays
        end: "0 20 * * 1-5"     # Scale down at 8 PM
        desiredReplicas: "10"
```

## Monitor Scaling Events

```bash
# Watch HPA scaling decisions
kubectl get hpa -n production -w

# Check Cluster Autoscaler logs
kubectl logs -n kube-system \
  -l app=cluster-autoscaler \
  --tail=50 | grep -i "scale"

# View scaling events
kubectl get events -n production \
  --field-selector reason=SuccessfulRescale \
  --sort-by='.lastTimestamp'

# KEDA scaler status
kubectl get scaledobjects -n production
kubectl describe scaledobject kafka-consumer-scaler -n production
```

## Conclusion

Automated cluster scaling in Rancher combines HPA for steady-state metric-based scaling, VPA for right-sizing pod resource requests, Cluster Autoscaler for node-level elasticity, and KEDA for event-driven workloads. Configure scale-down conservatively (longer stabilization windows) to avoid flapping, and use KEDA for queue-based workloads that need more precise scaling than CPU/memory metrics provide. Monitor scaling events through Grafana dashboards to validate that the scaling parameters match actual traffic patterns.
