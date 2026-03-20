# How to Automate Cluster Scaling in Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Autoscaling, Cluster-autoscaler, Kubernetes, Automation

Description: A guide to automating cluster scaling in Rancher using the Cluster Autoscaler, node pool scaling via the API, and HPA for workload scaling.

## Overview

Kubernetes provides multiple layers of autoscaling: Horizontal Pod Autoscaler (HPA) scales workloads, and Cluster Autoscaler scales the underlying node infrastructure. Rancher integrates with both cloud-provider Cluster Autoscalers and supports manual/API-based node pool scaling for on-premises environments. This guide covers setting up automated cluster scaling for Rancher-managed clusters.

## Horizontal Pod Autoscaling

HPA scales application pods based on CPU, memory, or custom metrics:

### Basic CPU-based HPA

```yaml
# HPA that scales based on CPU utilization

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
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
```

### HPA with Custom Metrics

```yaml
# HPA using custom Prometheus metric (requests per second)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa-custom
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"   # 1000 RPS per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0     # Scale up immediately
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60    # Add max 5 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scale down
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60    # Remove max 2 pods per minute
```

## Cluster Autoscaler for AWS (EKS on Rancher)

```yaml
# Cluster Autoscaler deployment for EKS clusters managed by Rancher
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --namespace=kube-system
            - --nodes=3:15:eks-worker-group-1   # min:max:ASG-name
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
            - --scale-down-utilization-threshold=0.5
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
          env:
            - name: AWS_REGION
              value: us-east-1
          resources:
            requests:
              cpu: 100m
              memory: 300Mi
```

## Scaling Rancher Node Pools via API

For on-premises or custom infrastructure, scale node pools through the Rancher API:

```bash
#!/bin/bash
# scale-nodepool.sh - Scale a Rancher node pool

RANCHER_URL="${RANCHER_URL}"
RANCHER_TOKEN="${RANCHER_TOKEN}"
CLUSTER_ID="$1"
NODEPOOL_ID="$2"
TARGET_QUANTITY="$3"

echo "Scaling node pool ${NODEPOOL_ID} to ${TARGET_QUANTITY} nodes..."

# Get current node pool configuration
NODEPOOL=$(curl -s -k \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  "${RANCHER_URL}/v3/nodepools/${NODEPOOL_ID}")

CURRENT_QUANTITY=$(echo "${NODEPOOL}" | jq -r '.quantity')
echo "Current quantity: ${CURRENT_QUANTITY}"

# Scale the node pool
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer ${RANCHER_TOKEN}" \
  -H "Content-Type: application/json" \
  "${RANCHER_URL}/v3/nodepools/${NODEPOOL_ID}" \
  -d "{\"quantity\": ${TARGET_QUANTITY}}"

echo "Scaling initiated. Waiting for nodes to be ready..."

# Wait for nodes to reach desired count
while true; do
  ACTIVE_NODES=$(curl -s -k \
    -H "Authorization: Bearer ${RANCHER_TOKEN}" \
    "${RANCHER_URL}/v3/nodes?nodePoolId=${NODEPOOL_ID}&state=active" \
    | jq '.data | length')

  echo "Active nodes: ${ACTIVE_NODES}/${TARGET_QUANTITY}"

  if [ "${ACTIVE_NODES}" -eq "${TARGET_QUANTITY}" ]; then
    echo "Node pool scaled successfully!"
    break
  fi
  sleep 30
done
```

## Scheduled Scaling with CronJobs

Scale clusters up before business hours and down on weekends:

```yaml
# CronJob: Scale up Monday morning
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-weekday
  namespace: automation
spec:
  schedule: "0 6 * * 1-5"    # 6 AM Mon-Fri
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
            - name: scaler
              image: registry.example.com/rancher-scaler:latest
              env:
                - name: RANCHER_URL
                  value: "https://rancher.example.com"
                - name: RANCHER_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: rancher-token
                      key: token
                - name: NODEPOOL_ID
                  value: "np-xxxxx"
                - name: TARGET_QUANTITY
                  value: "10"
              command: ["/scripts/scale-nodepool.sh"]
          restartPolicy: OnFailure
---
# CronJob: Scale down Friday evening
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-weekend
  namespace: automation
spec:
  schedule: "0 19 * * 5"    # 7 PM Friday
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler-sa
          containers:
            - name: scaler
              image: registry.example.com/rancher-scaler:latest
              env:
                - name: TARGET_QUANTITY
                  value: "3"    # Weekend minimum
              command: ["/scripts/scale-nodepool.sh"]
          restartPolicy: OnFailure
```

## Vertical Pod Autoscaler

VPA adjusts resource requests/limits automatically:

```yaml
# Install VPA (if not already installed)
# kubectl apply -f https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler

# VPA for database workload - auto-tune resource requests
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgresql
  updatePolicy:
    updateMode: Auto    # Automatically update pods
  resourcePolicy:
    containerPolicies:
      - containerName: postgresql
        minAllowed:
          cpu: 500m
          memory: 2Gi
        maxAllowed:
          cpu: 8
          memory: 32Gi
```

## Scaling Metrics and Alerts

```yaml
# Alert if cluster is near capacity
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-capacity-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: cluster-capacity
      rules:
        - alert: ClusterNearCapacity
          expr: |
            sum(kube_node_status_allocatable{resource="cpu"}) -
            sum(kube_pod_container_resource_requests{resource="cpu"})
            < 4
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Cluster has less than 4 CPU cores available"
```

## Conclusion

Automating cluster scaling in Rancher requires multiple complementary mechanisms: HPA for workload scaling, Cluster Autoscaler for node-level scaling in cloud environments, Rancher API scripts for on-premises scaling, and CronJobs for predictable schedule-based scaling. VPA provides intelligent resource tuning. Combine these with Prometheus alerting to ensure your clusters scale before performance degradation occurs, and scale down to save costs during low-traffic periods.
