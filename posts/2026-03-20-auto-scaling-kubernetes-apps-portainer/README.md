# How to Configure Auto-Scaling for Kubernetes Apps in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, HPA, Auto-Scaling, Performance

Description: Learn how to enable Horizontal Pod Autoscaler (HPA) for Kubernetes applications deployed through Portainer.

## What Is Horizontal Pod Autoscaling?

The Horizontal Pod Autoscaler (HPA) automatically adjusts the number of pod replicas based on observed metrics such as CPU utilization or memory usage. When load increases, HPA scales up; when load decreases, it scales down.

## Prerequisites

- Metrics Server must be installed in the cluster for CPU/memory-based autoscaling.

```bash
# Install Metrics Server (if not already installed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify Metrics Server is working
kubectl top nodes
```

## Enabling Auto-Scaling in Portainer

When deploying an application:

1. Scroll to the **Auto-scaling** section.
2. Toggle **Enable auto-scaling** to On.
3. Configure:
   - **Minimum replicas**: The floor for scaling down.
   - **Maximum replicas**: The ceiling for scaling up.
   - **Target CPU utilization**: Percentage at which to scale up (e.g., 70%).
4. Click **Deploy**.

## What Portainer Creates

Portainer creates an HPA resource targeting your Deployment:

```yaml
# HPA created by Portainer
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app          # Target deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale up when CPU > 70%
```

## Adding Memory-Based Scaling

```yaml
# Add memory scaling alongside CPU
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
        averageUtilization: 80  # Scale up when memory > 80%
```

## Managing HPA via CLI

```bash
# Create an HPA targeting a deployment
kubectl autoscale deployment my-app \
  --min=2 --max=10 --cpu-percent=70 \
  --namespace=production

# View HPA status and current replica count
kubectl get hpa --namespace production

# Describe HPA for detailed scaling events
kubectl describe hpa my-app-hpa --namespace production

# Delete an HPA
kubectl delete hpa my-app-hpa --namespace production
```

## Scaling Behavior Tuning

Prevent rapid scale-down with stabilization windows:

```yaml
spec:
  behavior:
    scaleDown:
      # Wait 5 minutes before scaling down
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1           # Remove at most 1 pod per step
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100         # Can double the replica count per step
          periodSeconds: 30
```

## Conclusion

Auto-scaling in Portainer enables your applications to handle variable load without manual intervention. Always set both CPU requests (required by HPA for calculations) and reasonable min/max replica bounds to prevent over-scaling.
