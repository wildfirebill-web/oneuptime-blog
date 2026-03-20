# How to Set Up Cost Management for Rancher Clusters - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cost-Management, Kubecost, Opencost, Kubernetes, FinOps

Description: A guide to implementing cost management for Rancher-managed Kubernetes clusters using OpenCost and Kubecost, including cost allocation, budgets, and optimization.

## Overview

Without proper cost visibility, Kubernetes infrastructure costs can spiral out of control. Rancher manages multiple clusters, making cross-cluster cost management essential. This guide covers deploying OpenCost or Kubecost on Rancher-managed clusters, configuring cost allocation by namespace and team, setting budgets, and identifying cost optimization opportunities.

## Why Kubernetes Cost Management Is Hard

- Shared infrastructure makes cost allocation non-trivial
- Unused resource reservations inflate costs
- Multiple clusters managed by Rancher span multiple accounts
- Developer self-service can lead to over-provisioning

## Step 1: Install OpenCost (Free, Open Source)

```bash
# Install OpenCost with Prometheus integration

# (Rancher Monitoring must already be installed)

helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm repo update

helm install opencost opencost/opencost \
  --namespace opencost \
  --create-namespace \
  --set opencost.prometheus.existingSecretName="" \
  --set opencost.prometheus.internal.enabled=false \
  --set opencost.prometheus.external.enabled=true \
  --set opencost.prometheus.external.url=http://rancher-monitoring-prometheus.cattle-monitoring-system.svc:9090

# Access OpenCost UI
kubectl port-forward svc/opencost 9090:9090 -n opencost
```

## Step 2: Configure Cloud Pricing (AWS)

```yaml
# AWS cloud pricing configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-costs
  namespace: opencost
data:
  cloud_integration.json: |
    {
      "aws": {
        "athenaBucketName": "my-cost-and-usage-report",
        "athenaRegion": "us-east-1",
        "athenaDatabase": "athenacurcfn",
        "athenaTable": "my_cur",
        "masterPayerARN": "arn:aws:iam::123456789012:role/CostAndUsageReportRole",
        "serviceKeyName": "aws_access_key_id",
        "serviceKeySecret": "aws_secret_access_key",
        "projectID": "123456789012"
      }
    }
```

## Step 3: Install Kubecost (More Features, Free Tier Available)

```bash
# Install Kubecost with Rancher integration
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update

helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set global.prometheus.fqdn=http://rancher-monitoring-prometheus.cattle-monitoring-system.svc:9090 \
  --set global.prometheus.enabled=false \
  --set cost-analyzer.nodeSelector."kubernetes.io/os"=linux

# Access Kubecost UI
kubectl port-forward svc/kubecost-cost-analyzer 9090:9090 -n kubecost
```

## Step 4: Cost Allocation Labels

Label your workloads for granular cost allocation:

```yaml
# Standard cost allocation labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
  labels:
    app: api-service
    team: platform-engineering    # Team label for allocation
    cost-center: "CC-1234"        # Finance cost center
    environment: production
    project: user-onboarding      # Project label
spec:
  template:
    metadata:
      labels:
        app: api-service
        team: platform-engineering
        cost-center: "CC-1234"
```

## Step 5: Budget Alerts

```yaml
# OpenCost Budget Alert (via Alertmanager)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-budget-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: cost-budgets
      rules:
        # Alert if a namespace exceeds $500/month
        - alert: NamespaceBudgetExceeded
          expr: |
            sum by(namespace) (
              opencost_container_cpu_cost_hourly +
              opencost_container_memory_cost_hourly
            ) * 24 * 30 > 500
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "Namespace {{ $labels.namespace }} exceeds $500/month budget"
            description: "Current monthly cost: ${{ $value | printf \"%.2f\" }}"

        # Alert if cluster monthly cost exceeds threshold
        - alert: ClusterCostSpike
          expr: |
            sum(opencost_container_cpu_cost_hourly + opencost_container_memory_cost_hourly) * 24 * 30 > 10000
          for: 2h
          labels:
            severity: critical
          annotations:
            summary: "Cluster monthly cost exceeds $10,000"
```

## Step 6: Identify Cost Optimization Opportunities

### Find Over-Provisioned Workloads

```bash
# Query Prometheus for CPU utilization vs requests
# Low utilization = over-provisioned = wasted money

kubectl top pods -A --sort-by=cpu | head -20

# Kubecost API to find savings opportunities
curl "http://kubecost.kubecost.svc:9090/model/savings" \
  | jq '.requestSizings[] | {name: .name, namespace: .namespace, monthlySavings: .monthlySavings}' \
  | sort -t: -k4 -rn \
  | head -20
```

### Identify Unused Volumes

```bash
# Find PVCs not attached to any pod
kubectl get pvc -A | grep -v Bound
kubectl get pv | grep Released

# Cost of unused PVCs (they still incur storage costs)
kubectl get pvc -A -o json \
  | jq -r '.items[] | select(.status.phase == "Bound") |
    {name: .metadata.name, namespace: .metadata.namespace, capacity: .spec.resources.requests.storage}' \
  | grep "Released"
```

### VPA Recommendations for Right-Sizing

```yaml
# Install VPA in recommendation mode (no auto-update)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  updatePolicy:
    updateMode: "Off"    # Recommendations only, no auto-update
```

```bash
# View VPA recommendations
kubectl get vpa api-service-vpa -n production -o json \
  | jq '.status.recommendation.containerRecommendations[0]'
# Shows: lowerBound, target, upperBound for CPU and memory
```

## Step 7: Cost Reports Dashboard

```bash
# Generate monthly cost report per team
curl "http://kubecost.kubecost.svc:9090/model/allocation" \
  --get \
  --data-urlencode "window=month" \
  --data-urlencode "aggregate=label:team" \
  --data-urlencode "accumulate=true" \
  | jq '.data[0] | to_entries[] | {team: .key, totalCost: .value.totalCost}' \
  | sort -t: -k4 -rn
```

## Step 8: Chargeback and Showback

```bash
#!/bin/bash
# generate-chargeback-report.sh
# Generate monthly chargeback report per team

MONTH="${1:-$(date +%Y-%m)}"

echo "Generating chargeback report for ${MONTH}"
echo ""
echo "Team,Namespace,CPU Cost,Memory Cost,Storage Cost,Total Cost"

# Query OpenCost API for namespace costs
curl -s "http://opencost.opencost.svc:9090/allocation/compute" \
  --get \
  --data-urlencode "window=${MONTH}" \
  --data-urlencode "aggregate=namespace" \
  | jq -r '.data[0] | to_entries[] |
    [
      (.value.properties.labels.team // "untagged"),
      .key,
      (.value.cpuCost | tostring),
      (.value.ramCost | tostring),
      (.value.pvCost | tostring),
      (.value.totalCost | tostring)
    ] | @csv'
```

## Conclusion

Cost management for Rancher clusters requires both visibility tools (OpenCost, Kubecost) and operational practices (labeling, budgeting, right-sizing). OpenCost provides a free, open-source foundation for cost visibility, while Kubecost adds features like savings recommendations and showback reports. Consistently applying cost allocation labels to all workloads is the most impactful action you can take for cost governance. Combine with VPA recommendations and regular right-sizing reviews to continuously optimize your Kubernetes infrastructure costs.
