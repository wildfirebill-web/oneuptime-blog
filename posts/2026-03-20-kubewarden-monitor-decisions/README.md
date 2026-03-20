# How to Monitor Kubewarden Policy Decisions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, Monitoring, Policy Decisions, Prometheus, Grafana, Kubernetes, SUSE Rancher

Description: Learn how to monitor Kubewarden policy admission decisions using Prometheus metrics, Grafana dashboards, and audit logs to track policy violations and cluster security posture.

---

Monitoring Kubewarden policy decisions helps you understand what is being allowed or denied in your cluster, detect misconfigured policies, and track your security posture over time.

---

## Kubewarden Metrics Overview

Kubewarden exports Prometheus metrics from each Policy Server. Key metrics:

| Metric | Description |
|---|---|
| `kubewarden_policy_evaluations_total` | Total policy evaluations by policy name and decision |
| `kubewarden_policy_evaluation_duration_seconds` | Policy evaluation latency |
| `kubewarden_policy_status` | Policy active status (1=active, 0=inactive) |

---

## Step 1: Enable Prometheus Metrics

Kubewarden's Policy Server exposes metrics on port 8080 by default. Create a ServiceMonitor:

```yaml
# kubewarden-servicemonitor.yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kubewarden-policy-server
  namespace: kubewarden
  labels:
    release: prometheus   # Must match your Prometheus operator selector
spec:
  selector:
    matchLabels:
      app: kubewarden-policy-server
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
  namespaceSelector:
    matchNames:
      - kubewarden
```

```bash
kubectl apply -f kubewarden-servicemonitor.yaml
```

---

## Step 2: Create Alerting Rules

```yaml
# kubewarden-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubewarden-alerts
  namespace: kubewarden
  labels:
    release: prometheus
spec:
  groups:
    - name: kubewarden.policies
      rules:
        - alert: KubewardenHighDenyRate
          expr: |
            (
              rate(kubewarden_policy_evaluations_total{decision="deny"}[5m])
              /
              rate(kubewarden_policy_evaluations_total[5m])
            ) > 0.10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Kubewarden policy '{{ $labels.policy_name }}' has >10% deny rate"
            description: "High deny rate may indicate a misconfigured policy or an attack"

        - alert: KubewardenPolicyError
          expr: rate(kubewarden_policy_evaluations_total{decision="error"}[5m]) > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Kubewarden policy '{{ $labels.policy_name }}' is returning errors"
```

---

## Step 3: Grafana Dashboard Queries

Create a Grafana dashboard with these queries:

```promql
# Policy evaluation rate by decision type
sum by (policy_name, decision) (
  rate(kubewarden_policy_evaluations_total[5m])
)

# Top 5 most active policies
topk(5, sum by (policy_name) (
  rate(kubewarden_policy_evaluations_total[5m])
))

# Policy evaluation latency (p95)
histogram_quantile(0.95,
  sum by (policy_name, le) (
    rate(kubewarden_policy_evaluation_duration_seconds_bucket[5m])
  )
)

# Deny rate per policy
sum by (policy_name) (
  rate(kubewarden_policy_evaluations_total{decision="deny"}[1h])
)
```

---

## Step 4: Enable Audit Scanner

Kubewarden's Audit Scanner runs periodic checks on existing resources to detect policy violations that were admitted before a policy was added:

```bash
# Check if audit scanner is installed
kubectl get deployment kubewarden-audit-scanner -n kubewarden

# Trigger a manual audit scan
kubectl create job manual-audit-$(date +%s) \
  --from=cronjob/kubewarden-audit-scanner \
  -n kubewarden

# View audit scan results
kubectl get policyreport -A
kubectl describe policyreport -A
```

---

## Step 5: View Policy Reports

Kubewarden uses the Policy Report CRD to store audit results:

```bash
# List all policy reports
kubectl get policyreport -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,FAIL:.summary.fail,PASS:.summary.pass'

# Get details of violations
kubectl get policyreport -A -o json | \
  jq '.items[].results[] | select(.result == "fail") | {policy:.policy, message:.message, resource:.subjects[0].name}'
```

---

## Best Practices

- Alert on error-level decisions - a policy returning `error` means it could not evaluate the admission request.
- Monitor for sudden spikes in deny rate - this could indicate a new policy blocking legitimate workloads.
- Use the Audit Scanner to identify existing resources that violate newly added policies before enforcing them.
