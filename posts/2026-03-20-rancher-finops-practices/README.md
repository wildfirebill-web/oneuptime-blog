# How to Configure FinOps Practices with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, finops, cost-optimization, kubernetes, cloud-economics

Description: A guide to implementing FinOps practices in Rancher environments, covering cost visibility, optimization strategies, accountability frameworks, and continuous cost governance.

## Overview

FinOps (Financial Operations) is a practice that brings together technology, business, and finance teams to enable data-driven cloud spending decisions. For organizations running Rancher with multiple Kubernetes clusters, FinOps practices help align costs with business outcomes and prevent cloud waste. This guide covers implementing the core FinOps phases — Inform, Optimize, and Operate — in Rancher environments.

## FinOps Framework for Kubernetes

The FinOps Framework defines three phases:

1. **Inform**: Gain visibility into cloud spending and usage
2. **Optimize**: Identify and act on cost reduction opportunities
3. **Operate**: Build sustainable cost governance processes

## Phase 1: Inform — Cost Visibility

### Mandatory Resource Tagging Policy

Enforce cost allocation tags via Kubewarden:

```yaml
# Policy: Require cost allocation labels on all Deployments
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: require-cost-labels
spec:
  module: registry://ghcr.io/kubewarden/policies/require-labels:v0.2.0
  rules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      resources: ["deployments", "statefulsets", "daemonsets"]
      operations: ["CREATE", "UPDATE"]
  settings:
    mandatory_labels:
      - team           # Engineering team owner
      - cost-center    # Finance cost center
      - environment    # production, staging, dev
      - project        # Business project/product
```

### Real-Time Cost Dashboard

```bash
# Query OpenCost for current spending
curl -s "http://opencost.opencost.svc:9090/allocation/compute" \
  --get \
  --data-urlencode "window=today" \
  --data-urlencode "aggregate=label:cost-center" \
  | jq '.data[0] | to_entries[] |
    {
      costCenter: .key,
      dailyCost: (.value.totalCost | round),
      monthlyProjected: ((.value.totalCost * 30) | round)
    }' \
  | sort -t: -k3 -rn
```

### Cross-Cluster Cost Aggregation

```python
#!/usr/bin/env python3
# aggregate-cluster-costs.py - Aggregate costs across all Rancher clusters

import requests
import json
from typing import Dict, List

OPENCOST_ENDPOINTS = {
    "prod-us-east": "http://prod-us-east-opencost.svc:9090",
    "prod-eu-west": "http://prod-eu-west-opencost.svc:9090",
    "staging": "http://staging-opencost.svc:9090"
}

def get_cluster_costs(cluster_name: str, endpoint: str) -> Dict:
    """Fetch costs from an OpenCost endpoint"""
    resp = requests.get(
        f"{endpoint}/allocation/compute",
        params={"window": "month", "aggregate": "label:team", "accumulate": "true"}
    )
    resp.raise_for_status()
    data = resp.json().get('data', [{}])[0]
    return {
        "cluster": cluster_name,
        "teams": {
            team: round(info.get('totalCost', 0), 2)
            for team, info in data.items()
        }
    }

def aggregate_costs(all_costs: List[Dict]) -> Dict:
    """Aggregate costs across all clusters"""
    aggregated = {}
    for cluster_data in all_costs:
        for team, cost in cluster_data['teams'].items():
            aggregated[team] = aggregated.get(team, 0) + cost
    return aggregated

if __name__ == '__main__':
    all_costs = []
    for cluster, endpoint in OPENCOST_ENDPOINTS.items():
        try:
            costs = get_cluster_costs(cluster, endpoint)
            all_costs.append(costs)
        except Exception as e:
            print(f"Failed to get costs from {cluster}: {e}")

    aggregated = aggregate_costs(all_costs)
    print("\n=== Monthly Cost by Team (All Clusters) ===")
    for team, cost in sorted(aggregated.items(), key=lambda x: x[1], reverse=True):
        print(f"  {team}: ${cost:,.2f}")
    print(f"\n  TOTAL: ${sum(aggregated.values()):,.2f}")
```

## Phase 2: Optimize — Cost Reduction

### Identify and Right-Size Over-Provisioned Workloads

```bash
#!/bin/bash
# find-oversized-workloads.sh - Find pods with high CPU overprovisioning

echo "Workloads with CPU utilization below 20% of requests:"
echo ""

# Use kubectl and metrics-server to find low-utilization pods
kubectl top pods -A --sort-by=cpu | while read namespace pod cpu memory; do
  # Skip header
  [ "$namespace" = "NAMESPACE" ] && continue

  # Get CPU requests
  REQUESTS=$(kubectl get pod "$pod" -n "$namespace" \
    -o jsonpath='{.spec.containers[0].resources.requests.cpu}' 2>/dev/null)

  if [ -n "$REQUESTS" ]; then
    echo "Pod: $namespace/$pod - Using: ${cpu}, Requested: ${REQUESTS}"
  fi
done
```

### Implement Namespace Resource Quotas

Prevent over-provisioning with ResourceQuotas per team:

```yaml
# ResourceQuota for each team namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-platform-eng
  annotations:
    cost-center: "CC-001"
    monthly-budget: "2000"   # USD annotation (informational)
spec:
  hard:
    # Compute
    requests.cpu: "20"
    limits.cpu: "40"
    requests.memory: "40Gi"
    limits.memory: "80Gi"
    # Storage
    persistentvolumeclaims: "20"
    requests.storage: "500Gi"
    # Object count
    pods: "100"
    services.loadbalancers: "5"
```

### Spot/Preemptible Node Scheduling

```yaml
# Toleration for spot instances - schedule batch workloads on cheaper spot nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
  namespace: batch
spec:
  template:
    spec:
      tolerations:
        - key: "cloud.google.com/gke-spot"
          operator: Exists
          effect: NoSchedule
        - key: "kubernetes.azure.com/scalesetpriority"
          value: spot
          operator: Equal
          effect: NoSchedule
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: cloud.google.com/gke-spot
                    operator: Exists
```

### Scale Down Non-Production Clusters at Night

```yaml
# CronJob: Scale down dev/staging clusters at 8 PM weekdays
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dev-cluster-scale-down
  namespace: finops
spec:
  schedule: "0 20 * * 1-5"    # 8 PM Mon-Fri
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: scaler
              image: registry.example.com/rancher-scaler:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Scale all deployments in dev namespaces to 0
                  for ns in $(kubectl get ns -l environment=development -o name); do
                    kubectl scale deployments --all --replicas=0 -n "${ns#namespace/}"
                  done
          restartPolicy: OnFailure
```

## Phase 3: Operate — Cost Governance

### Monthly FinOps Review Process

```bash
#!/bin/bash
# finops-monthly-review.sh

MONTH=$(date -d "last month" +%Y-%m)

echo "=========================================="
echo "FinOps Monthly Review: ${MONTH}"
echo "=========================================="

# 1. Generate team cost report
echo ""
echo "1. Cost by Team:"
curl -s "http://opencost.opencost.svc:9090/allocation/compute" \
  --get \
  --data-urlencode "window=${MONTH}" \
  --data-urlencode "aggregate=label:team" \
  --data-urlencode "accumulate=true" \
  | jq -r '.data[0] | to_entries[] | "   \(.key): $\(.value.totalCost | round)"'

# 2. Compare to previous month (manual calculation in full implementation)
echo ""
echo "2. Month-over-month change: (see cost dashboard)"

# 3. List top 10 most expensive namespaces
echo ""
echo "3. Top 10 Most Expensive Namespaces:"
curl -s "http://opencost.opencost.svc:9090/allocation/compute" \
  --get \
  --data-urlencode "window=${MONTH}" \
  --data-urlencode "aggregate=namespace" \
  --data-urlencode "accumulate=true" \
  | jq -r '[.data[0] | to_entries[] | {ns: .key, cost: .value.totalCost}] | sort_by(-.cost) | .[0:10][] | "   \(.ns): $\(.cost | round)"'
```

### Cost Anomaly Detection

```yaml
# Alert on unexpected cost increases
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cost-anomaly-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: finops-anomaly
      rules:
        - alert: UnexpectedCostIncrease
          expr: |
            # Alert if hourly cost increases by more than 50% vs previous hour
            (
              sum(opencost_container_cpu_cost_hourly + opencost_container_memory_cost_hourly)
              /
              sum(opencost_container_cpu_cost_hourly + opencost_container_memory_cost_hourly offset 1h)
            ) > 1.5
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "Kubernetes cost increased by more than 50% in the last hour"
```

### FinOps KPI Dashboard

```yaml
# Grafana dashboard ConfigMap for FinOps KPIs
apiVersion: v1
kind: ConfigMap
metadata:
  name: finops-dashboard
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"
data:
  finops-kpis.json: |
    {
      "title": "FinOps KPIs Dashboard",
      "panels": [
        {"title": "Monthly Spend", "type": "stat"},
        {"title": "Cost per Team", "type": "piechart"},
        {"title": "CPU Utilization vs Requests", "type": "gauge"},
        {"title": "Cost Trend (90 days)", "type": "timeseries"}
      ]
    }
```

## FinOps Maturity Assessment

| Practice | Crawl | Walk | Run |
|---|---|---|---|
| Cost visibility | Basic Prometheus | OpenCost per cluster | Cross-cluster aggregation |
| Tagging | Some labels | Enforced via policy | Full allocation model |
| Budgets | No budgets | Manual alerts | Automated enforcement |
| Optimization | Ad-hoc | Monthly review | Continuous VPA + spot |
| Accountability | IT only | IT + Finance | Engineering + Finance + Product |

## Conclusion

FinOps in Rancher environments requires combining tooling (OpenCost, Kubecost), policies (mandatory labels, ResourceQuotas), and processes (monthly reviews, chargeback reports). The three phases — Inform, Optimize, and Operate — provide a roadmap from basic cost visibility to continuous cost optimization. Start with mandatory cost allocation labels enforced via Kubewarden, deploy OpenCost for visibility, and schedule regular FinOps review meetings with engineering and finance stakeholders. Cost optimization is an ongoing practice, not a one-time project.
