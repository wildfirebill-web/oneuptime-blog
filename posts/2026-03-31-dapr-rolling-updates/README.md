# How to Implement Rolling Updates for Dapr Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Rolling Update, Deployment, Kubernetes, DevOps

Description: Learn how to implement rolling updates for Dapr applications in Kubernetes, including sidecar version coordination, readiness probes, and zero-downtime update strategies.

---

Rolling updates deploy new versions of a Dapr application gradually, replacing old pods one at a time while maintaining availability. Dapr applications have an additional consideration: the application container and the Dapr sidecar must work together correctly during the transition.

## Configure Rolling Update Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/config: "dapr-config"
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: order-service
        image: myrepo/order-service:2.1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
```

## Readiness Endpoint in Application Code

The readiness probe must confirm both the app and Dapr sidecar are ready:

```python
from flask import Flask, jsonify
import requests

app = Flask(__name__)

@app.route('/health/ready', methods=['GET'])
def readiness():
    # Check that Dapr sidecar is responsive
    try:
        resp = requests.get(
            'http://localhost:3500/v1.0/healthz/outbound',
            timeout=2
        )
        if resp.status_code != 204:
            return jsonify({"status": "not ready", "reason": "Dapr sidecar not ready"}), 503
    except requests.RequestException as e:
        return jsonify({"status": "not ready", "reason": str(e)}), 503

    return jsonify({"status": "ready"}), 200

@app.route('/health/live', methods=['GET'])
def liveness():
    return jsonify({"status": "alive"}), 200
```

## Performing the Rolling Update

```bash
# Update the image tag to trigger a rolling update
kubectl set image deployment/order-service \
  order-service=myrepo/order-service:2.2.0 \
  -n production

# Watch the rollout progress
kubectl rollout status deployment/order-service -n production --timeout=10m

# Monitor pod transitions
kubectl get pods -n production -l app=order-service -w
```

## Coordinating Dapr Sidecar Upgrades

When upgrading the Dapr control plane, the sidecars need to be restarted to pick up the new version:

```bash
# Upgrade Dapr control plane
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --version 1.14.0 \
  --wait

# Restart all Dapr-annotated deployments to update sidecars
kubectl get deployments --all-namespaces \
  -o jsonpath='{range .items[?(@.spec.template.metadata.annotations.dapr\.io/enabled=="true")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}' | \
  while read NS_NAME; do
    NAMESPACE=$(echo "$NS_NAME" | cut -d'/' -f1)
    NAME=$(echo "$NS_NAME" | cut -d'/' -f2)
    kubectl rollout restart deployment/"$NAME" -n "$NAMESPACE"
  done
```

## Annotating for Canary Updates

```bash
# Deploy canary version to 1 replica while keeping stable at 2
kubectl scale deployment/order-service --replicas=2 -n production
kubectl apply -f order-service-canary.yaml  # single replica with new image

# Gradually shift traffic
kubectl scale deployment/order-service-canary --replicas=2 -n production
kubectl scale deployment/order-service --replicas=1 -n production
```

## Summary

Rolling updates for Dapr applications require proper readiness probes that check both application health and Dapr sidecar availability before routing traffic to new pods. Set `maxUnavailable: 0` to ensure capacity is maintained during updates. When upgrading Dapr's control plane, restart deployments afterward to update sidecar versions without application downtime.
