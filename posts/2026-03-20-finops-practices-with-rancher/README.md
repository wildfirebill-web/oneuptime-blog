# How to Configure FinOps Practices with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, FinOps, Cost Optimization, Cloud, Kubernetes, Chargeback

Description: Implement FinOps practices with Rancher including cost allocation, chargeback/showback reporting, rightsizing automation, budget enforcement, and building a culture of cost awareness across...

## Introduction

FinOps (Financial Operations) for Kubernetes applies financial accountability principles to cloud infrastructure. In Rancher environments, FinOps means measuring where money is spent (cost allocation), attributing costs to responsible teams (chargeback/showback), optimizing spending (rightsizing), and creating incentives for cost-conscious engineering. This guide covers implementing the FinOps framework for Rancher multi-cluster deployments.

## FinOps Maturity Levels for Kubernetes

| Level | Capability |
|---|---|
| Crawl | Visibility: see costs per cluster |
| Walk | Allocation: costs per namespace/team + showback |
| Run | Optimization: automated rightsizing + budget enforcement |

## Step 1: Cost Allocation Labels

Consistent labels are the foundation of cost allocation:

```yaml
# Labeling standard - enforce via admission webhook

# Every deployment MUST have these labels:

metadata:
  labels:
    team: "payments"              # Team responsible
    cost-center: "CC-1042"        # Finance cost center
    environment: "production"     # prod/staging/dev
    product: "checkout-service"   # Business product
    application: "api"            # Technical component

# LimitRange to ensure resources have requests (required for cost allocation)
apiVersion: v1
kind: LimitRange
metadata:
  name: cost-allocation-requirements
  namespace: payments-prod
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

## Step 2: Chargeback Reports with OpenCost

```python
# generate_chargeback_report.py

import requests
import pandas as pd
from datetime import datetime, timedelta

OPENCOST_URL = "http://opencost.opencost.svc:9003"

def get_team_costs(window: str = "30d") -> pd.DataFrame:
    """Get costs aggregated by team label"""
    resp = requests.get(
        f"{OPENCOST_URL}/model/allocation/query",
        params={
            'window': window,
            'aggregate': 'label:team',
            'accumulate': 'true'
        }
    )
    data = resp.json()

    rows = []
    for allocation_set in data['data']:
        for team, allocation in allocation_set['sets'][0]['allocations'].items():
            rows.append({
                'team': team,
                'cpu_cost': round(allocation['cpuCost'], 2),
                'memory_cost': round(allocation['ramCost'], 2),
                'storage_cost': round(allocation['pvCost'], 2),
                'network_cost': round(allocation['networkCost'], 2),
                'total_cost': round(allocation['totalCost'], 2),
                'cpu_efficiency': round(allocation.get('cpuEfficiency', 0) * 100, 1),
                'memory_efficiency': round(allocation.get('ramEfficiency', 0) * 100, 1)
            })

    return pd.DataFrame(rows).sort_values('total_cost', ascending=False)

# Generate monthly chargeback
df = get_team_costs('30d')
print(df.to_string(index=False))

# Export to finance team
df.to_csv(f'chargeback-{datetime.now().strftime("%Y-%m")}.csv', index=False)
```

## Step 3: Budget Enforcement

```yaml
# ResourceQuota as budget guard rail
# Calculate max resources based on budget
# $500/month budget → ~16 vCPU + 32GB RAM at $0.031/vCPU-hr

apiVersion: v1
kind: ResourceQuota
metadata:
  name: budget-500-monthly
  namespace: team-frontend
  annotations:
    monthly-budget-usd: "500"
    cost-center: "CC-2024"
spec:
  hard:
    requests.cpu: "16"        # $500 ÷ ($0.031 × 730h) ≈ 22 vCPU
    requests.memory: "32Gi"   # $500 ÷ ($0.004 × 730h) ≈ 171 GB
    # Conservative limits to leave headroom
```

```yaml
# Rancher Project-level budget controls
# Configure via Rancher UI: Cluster > Projects > {Project} > Resource Quotas

# Or via Rancher API:
apiVersion: management.cattle.io/v3
kind: Project
metadata:
  name: frontend-team
spec:
  displayName: "Frontend Team"
  resourceQuota:
    limit:
      limitsCpu: "32"
      limitsMemory: "64Gi"
      requestsStorage: "500Gi"
```

## Step 4: Rightsizing Automation

```bash
#!/bin/bash
# rightsize_report.sh - Identify overprovisioned workloads

echo "=== Rightsizing Report ==="
echo "Generated: $(date)"
echo ""

kubectl get namespaces -o name | while read ns; do
  NS=${ns#namespace/}
  # Skip system namespaces
  [[ "$NS" == kube-* || "$NS" == cattle-* || "$NS" == rancher-* ]] && continue

  # Get VPA recommendations
  vpas=$(kubectl get vpa -n "$NS" -o json 2>/dev/null)
  if [ -z "$vpas" ]; then continue; fi

  echo "$vpas" | jq -r --arg ns "$NS" '
    .items[] |
    .metadata.name as $name |
    .status.recommendation.containerRecommendations[]? |
    "Namespace: \($ns) | Workload: \($name) | Container: \(.containerName) | Recommended CPU: \(.target.cpu) | Recommended Memory: \(.target.memory)"
  '
done
```

## Step 5: Spot/Preemptible Instances for Non-Prod

```yaml
# Node pool for spot instances (dev/test workloads)
apiVersion: provisioning.cattle.io/v1
kind: Cluster
spec:
  rkeConfig:
    machinePools:
      # On-demand for production workloads
      - name: prod-workers
        quantity: 5
        roles: [worker]
        nodeConfig:
          instanceType: m5.2xlarge
          spotPrice: ""    # On-demand

      # Spot for dev/test (60-80% cost reduction)
      - name: dev-spot-workers
        quantity: 10
        roles: [worker]
        nodeConfig:
          instanceType: m5.2xlarge
          spotPrice: "0.15"    # Bid price
        labels:
          workload-type: "non-critical"
        taints:
          - key: spot
            value: "true"
            effect: NoSchedule
```

## Step 6: FinOps Dashboard

```yaml
# Grafana dashboard panels for FinOps visibility
panels:
  - title: "Monthly Spend by Team"
    type: barchart
    targets:
      - expr: |
          sum by (label_team) (
            container_cpu_allocation{label_team!=""} * on(node) group_left()
            node_cpu_hourly_cost
          ) * 730

  - title: "Cost Efficiency by Namespace"
    type: table
    # Shows: actual usage vs. requested (waste percentage)
    targets:
      - expr: |
          (
            sum by (namespace) (rate(container_cpu_usage_seconds_total[1h]))
            /
            sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
          ) * 100
        legendFormat: "CPU Efficiency %"

  - title: "Idle Resources Cost"
    type: stat
    targets:
      - expr: |
          sum(kube_pod_container_resource_requests{resource="cpu"})
          - sum(rate(container_cpu_usage_seconds_total[1h]))
          # × hourly cost × 730 hours
```

## FinOps Maturity Checklist

- Labels: team, cost-center, environment on all resources
- OpenCost deployed with cloud provider pricing
- Monthly chargeback report delivered to finance
- Budget quotas enforced via ResourceQuota
- VPA recommendations reviewed and applied quarterly
- Spot instances for dev/test (target: 50% of non-prod on spot)
- Idle resources reviewed and cleaned up monthly
- FinOps review meeting with team leads monthly
- Cost metrics in team-level Grafana dashboards

## Conclusion

FinOps with Rancher transforms Kubernetes cost management from invisible to accountable. The key steps are consistent labeling for allocation, OpenCost for measurement, chargeback reports for team accountability, and ResourceQuotas for budget enforcement. The biggest cultural shift is making teams see their own costs-once teams see that a forgotten development deployment costs $500/month, they clean it up. Monthly FinOps review meetings with cost data create the organizational feedback loop that drives continuous optimization.
