# How to Configure Progressive Delivery with Flagger on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Progressive Delivery, Flagger, Canary, Blue-Green, Kubernetes

Description: Configure progressive delivery in Rancher using Flagger for automated canary releases and blue-green deployments with metric-based promotion and automatic rollback on SLO violations.

## Introduction

Progressive delivery extends continuous delivery by automatically controlling how much traffic reaches a new deployment version, using metrics to validate the release before full rollout. Flagger is a CNCF Kubernetes operator that automates canary releases and blue-green deployments, integrating with NGINX Ingress, Istio, or Linkerd for traffic splitting and Prometheus for metric-based gating.

## Step 1: Install Flagger

```bash
# Install Flagger with NGINX Ingress provider
helm repo add flagger https://flagger.app
helm repo update

helm install flagger flagger/flagger \
  --namespace flagger-system \
  --create-namespace \
  --set meshProvider=nginx \
  --set metricsServer=http://prometheus.monitoring.svc:9090

# Install Flagger load tester (generates test traffic)
helm install flagger-loadtester flagger/loadtester \
  --namespace flagger-system
```

## Step 2: Configure Canary Release

```yaml
# Canary resource for automatic progressive rollout
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api-server
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  # Service configuration
  service:
    port: 80
    targetPort: 8080
    gateways:
      - public
    hosts:
      - api.company.com

  # Progressive rollout configuration
  progressDeadlineSeconds: 600
  canaryAnalysis:
    # Time between traffic weight increments
    interval: 1m

    # Traffic increment per step
    stepWeight: 10

    # Maximum canary traffic weight
    maxWeight: 50

    # How long to wait before rolling back
    threshold: 5     # 5 failed checks → rollback

    # Metrics to evaluate at each step
    metrics:
      - name: request-success-rate
        # Fail if success rate < 99%
        thresholdRange:
          min: 99
        interval: 1m

      - name: request-duration
        # Fail if P99 > 500ms
        thresholdRange:
          max: 500
        interval: 1m

    # Webhooks (optional: run integration tests during rollout)
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.flagger-system/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sf http://api-server-canary.production/health"

      - name: load-test
        url: http://flagger-loadtester.flagger-system/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://api-server-canary.production/"
```

## Step 3: Monitor Canary Progress

```bash
# Watch canary progression
kubectl describe canary api-server -n production

# Example output during rollout:
# Status:
#   Canary Weight: 30         ← Currently sending 30% traffic to canary
#   Failed Checks: 0
#   Phase: Progressing
#   Conditions:
#   - Status: True
#     Type: Promoted

# Watch events
kubectl get events -n production \
  --field-selector involvedObject.name=api-server \
  --sort-by=.lastTimestamp -w

# Check metrics
kubectl get canary api-server -n production \
  -o jsonpath='{.status}'
```

## Step 4: Blue-Green Deployment

```yaml
# Blue-green with instant traffic switch (vs. incremental canary)
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api-server-bg
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  service:
    port: 80

  canaryAnalysis:
    # Blue-green: one big test then promote or rollback
    stepWeight: 100         # Send 100% traffic to new version immediately
    threshold: 5
    interval: 30s

    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99.5
        interval: 30s

    # Keep old (blue) version for quick rollback
    # Flagger keeps the old deployment until promotion confirmed
```

## Step 5: Custom Metric Templates

```yaml
# Create custom Prometheus metric template
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: error-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring.svc:9090
  query: |
    100 - sum(
      rate(http_requests_total{
        namespace="{{ namespace }}",
        service="{{ target }}",
        status!~"5.."
      }[{{ interval }}])
    )
    /
    sum(
      rate(http_requests_total{
        namespace="{{ namespace }}",
        service="{{ target }}"
      }[{{ interval }}])
    ) * 100
```

## Step 6: Rollback Scenarios

```bash
# Manual rollback if needed
kubectl annotate canary api-server \
  flagger.app/rollback="true" \
  -n production

# Automatic rollback triggers:
# - Success rate drops below 99%
# - P99 latency exceeds 500ms
# - 5 consecutive metric check failures

# View rollback event
kubectl get events -n production \
  --field-selector reason=Rolled_Back

# After rollback, the deployment returns to original version
# Canary is destroyed, primary (stable) remains
```

## Step 7: Notifications

```yaml
# Slack notifications for Flagger events
apiVersion: flagger.app/v1beta1
kind: AlertProvider
metadata:
  name: slack-platform
  namespace: flagger-system
spec:
  type: slack
  channel: "#deployments"
  username: Flagger
  secretRef:
    name: slack-alert-provider
---
# Attach alert provider to canary
# Add to Canary spec:
spec:
  canaryAnalysis:
    alerts:
      - name: "Platform team alerts"
        severity: warn
        providerRef:
          name: slack-platform
          namespace: flagger-system
```

## Conclusion

Flagger on Rancher automates the riskiest part of software delivery: promoting new versions to production. Canary releases gradually shift traffic while continuously validating success rate and latency metrics. If any metric violates the defined SLO threshold, Flagger automatically rolls back to the stable version without human intervention. This transforms deployment from a high-risk manual event into a safe, automated progression that teams can trigger from CI/CD pipelines with confidence.
