# How to Monitor Rancher HA Cluster Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, HA, Health Monitoring, Prometheus, Alerting

Description: Monitor the health of your Rancher HA deployment with comprehensive checks for etcd, API server, Rancher pods, and managed cluster connectivity.

## Introduction

Proactive health monitoring of your Rancher HA deployment allows you to detect and address issues before they impact operations. This guide covers health checks at every layer: etcd health, RKE2 API server, Rancher server pods, load balancer, and managed cluster connectivity.

## Prerequisites

- Running Rancher HA deployment
- Prometheus and Grafana (via rancher-monitoring)
- AlertManager configured with notification channels
- PagerDuty, Slack, or email for alerts

## Step 1: etcd Health Monitoring

```bash
# Continuous etcd health check script
#!/bin/bash
ETCDCTL_OPTS="--endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key"

# Health check
etcdctl endpoint health $ETCDCTL_OPTS

# Status (includes DB size, raft index)
etcdctl endpoint status $ETCDCTL_OPTS -w table

# Performance check
etcdctl check perf $ETCDCTL_OPTS

# Member list
etcdctl member list $ETCDCTL_OPTS -w table
```

## Step 2: Configure etcd Health Alerts

```yaml
# etcd-alerts.yaml - Critical etcd alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-health-alerts
  namespace: cattle-monitoring-system
  labels:
    release: rancher-monitoring
spec:
  groups:
    - name: etcd.health
      rules:
        - alert: EtcdMembersDown
          expr: max without (endpoint) (etcd_server_has_leader) < 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "etcd has no leader"
            description: "etcd cluster has lost quorum"

        - alert: EtcdNoLeader
          expr: etcd_server_has_leader == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "etcd member has no leader"

        - alert: EtcdHighNumberOfLeaderChanges
          expr: rate(etcd_server_leader_changes_seen_total[15m]) > 3
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "etcd leader changes too frequently"

        - alert: EtcdDatabaseSizeLimitApproaching
          expr: |
            etcd_mvcc_db_total_size_in_bytes /
            etcd_server_quota_backend_bytes > 0.8
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd database >80% of quota"

        - alert: EtcdGRPCRequestsSlow
          expr: |
            histogram_quantile(0.99,
              rate(grpc_server_handling_seconds_bucket{
                grpc_type="unary"
              }[5m])
            ) > 0.15
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "etcd gRPC requests slow (p99 > 150ms)"
```

## Step 3: Monitor Rancher Server Health

```yaml
# rancher-ha-health-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rancher-ha-health
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: rancher.ha
      rules:
        # Rancher pod availability
        - alert: RancherPodsUnavailable
          expr: |
            kube_deployment_status_replicas_unavailable{
              namespace="cattle-system",
              deployment="rancher"
            } > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Rancher pods unavailable"
            description: "{{ $value }} Rancher pods are unavailable"

        # Rancher pod count below expected
        - alert: RancherPodsInsufficient
          expr: |
            kube_deployment_status_replicas_ready{
              namespace="cattle-system",
              deployment="rancher"
            } < 2
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Fewer than 2 Rancher pods ready"

        # Rancher OOMKilled
        - alert: RancherOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{
              namespace="cattle-system",
              reason="OOMKilled"
            } == 1
          labels:
            severity: critical
          annotations:
            summary: "Rancher pod OOMKilled - increase memory limits"
```

## Step 4: Monitor Managed Cluster Connectivity

```yaml
# managed-cluster-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: managed-cluster-health
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: rancher.clusters
      rules:
        - alert: ManagedClusterDisconnected
          expr: rancher_cluster_ready == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Managed cluster {{ $labels.display_name }} disconnected"

        - alert: ManyManagedClustersDisconnected
          expr: sum(rancher_cluster_ready == 0) > 5
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "{{ $value }} managed clusters disconnected (possible Rancher issue)"
```

## Step 5: Health Check Dashboard (Grafana)

```bash
# Import Rancher HA health dashboard
# Dashboard JSON available at: https://grafana.com/grafana/dashboards/

# Key panels to include:
# 1. etcd cluster health status
# 2. etcd database size % used
# 3. etcd leader changes rate
# 4. Rancher pod availability
# 5. Rancher API latency (p50, p99)
# 6. Active cluster connections (WebSockets)
# 7. Managed cluster health status
# 8. Load balancer connection counts
```

## Step 6: External Health Check

```bash
# Run external health check from outside the cluster
# This validates the full stack (LB -> Rancher -> etcd)

#!/bin/bash
RANCHER_URL="https://rancher.example.com"
ALERT_EMAIL="ops@company.com"

check_rancher_health() {
    HTTP_STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "$RANCHER_URL/ping")
    RESPONSE_TIME=$(curl -sk -o /dev/null -w "%{time_total}" "$RANCHER_URL/ping")

    if [ "$HTTP_STATUS" != "200" ]; then
        echo "CRITICAL: Rancher returned $HTTP_STATUS"
        return 1
    fi

    if (( $(echo "$RESPONSE_TIME > 5.0" | bc -l) )); then
        echo "WARNING: Rancher response time ${RESPONSE_TIME}s > 5s threshold"
        return 2
    fi

    echo "OK: Rancher healthy (${RESPONSE_TIME}s)"
    return 0
}

# Run check
check_rancher_health
```

## Step 7: Recovery Runbook

```bash
# Automated recovery checks and actions

#!/bin/bash
# Rancher HA health check and recovery

# Check 1: etcd quorum
ETCD_HEALTHY=$(kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key 2>&1 | grep "is healthy" | wc -l)

if [ "$ETCD_HEALTHY" -lt "2" ]; then
    echo "CRITICAL: Less than 2 etcd members healthy"
    # Page on-call
fi

# Check 2: Rancher pods
RANCHER_READY=$(kubectl get deployment rancher -n cattle-system \
  -o jsonpath='{.status.readyReplicas}')

if [ "$RANCHER_READY" -lt "2" ]; then
    echo "WARNING: Only $RANCHER_READY Rancher replicas ready"
fi
```

## Conclusion

Comprehensive Rancher HA health monitoring requires visibility into multiple layers: etcd cluster health, RKE2 API server availability, Rancher pod status, and managed cluster connectivity. Alert on conditions that precede failures (etcd database size approaching quota, leader election frequency) rather than just reacting to outages. The combination of internal Prometheus alerts and external synthetic health checks provides defense-in-depth monitoring that catches issues regardless of whether the monitoring system itself is affected.
