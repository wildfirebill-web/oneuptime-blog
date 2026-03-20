# How to Optimize Rancher Server Performance - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Performance, Optimization, Server Tuning, Kubernetes

Description: Optimize Rancher server performance through proper resource allocation, database tuning, API rate limiting, and caching configurations for large-scale deployments.

## Introduction

As Rancher deployments grow to manage dozens or hundreds of clusters, server performance becomes critical. This guide covers key optimization techniques including resource sizing, database tuning, API performance, and configuration best practices for high-performance Rancher deployments.

## Prerequisites

- Running Rancher installation (v2.7+)
- Access to the Kubernetes cluster hosting Rancher
- Monitoring stack (Prometheus/Grafana) for performance analysis

## Step 1: Right-Size Rancher Server Resources

```yaml
# rancher-values.yaml - Optimized Helm values for Rancher

replicas: 3  # HA deployment

resources:
  requests:
    cpu: 2000m
    memory: 4Gi
  limits:
    cpu: 4000m
    memory: 8Gi

# Increase JVM heap for Norman (Rancher's API server)
extraEnv:
  - name: JAVA_OPTS
    value: "-Xms2g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

  # Reduce reconciliation frequency for large deployments
  - name: CATTLE_RESYNC_DEFAULT
    value: "900"  # 15 minutes instead of default 5

  # Increase goroutine limits
  - name: GOMAXPROCS
    value: "8"
```

## Step 2: Optimize etcd for Rancher's Local Cluster

```yaml
# rke2-config.yaml - Tuned etcd for Rancher's local cluster
etcd:
  # Increase snapshot frequency for faster recovery
  snapshotScheduleCron: "0 */4 * * *"
  snapshotRetention: 5

  # Increase etcd quotas for large state
  extraArgs:
    quota-backend-bytes: "8589934592"  # 8GB
    auto-compaction-mode: "periodic"
    auto-compaction-retention: "1h"
    max-request-bytes: "10485760"  # 10MB
    heartbeat-interval: "100"
    election-timeout: "1000"
```

```bash
# Defragment etcd regularly
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=etcd -o name | head -1) \
  -- etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key
```

## Step 3: Configure External Database for HA

```yaml
# rancher-ha-values.yaml - External MySQL for Rancher (deprecated in newer versions)
# For Rancher v2.6+, the local RKE2 cluster's etcd is the primary store
# But for older deployments:

externalTLS:
  enabled: true

# Database URL (if using external DB)
databaseURL: "mysql://rancher:password@mysql.example.com:3306/rancher?tls=true"
```

## Step 4: Tune Rancher API Server

```bash
# Set API rate limits via Helm
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set extraEnv[0].name=CATTLE_SERVER_URL \
  --set extraEnv[0].value=https://rancher.example.com \
  --set extraEnv[1].name=CATTLE_FEATURES \
  --set extraEnv[1].value="fleet=true,monitoringV2=true"
```

## Step 5: Configure Horizontal Pod Autoscaler

```yaml
# rancher-hpa.yaml - Scale Rancher pods based on load
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rancher
  namespace: cattle-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rancher
  minReplicas: 3
  maxReplicas: 5
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
          averageUtilization: 75
```

## Step 6: Monitor Rancher Server Metrics

```bash
# Check Rancher pod resource usage
kubectl top pods -n cattle-system

# Watch Rancher API latency
kubectl logs -n cattle-system \
  deployment/rancher \
  --since=1h | grep "slow request"

# Check Rancher audit log for expensive operations
kubectl logs -n cattle-system \
  deployment/rancher-audit-log \
  | jq '.responseStatus.code,.requestURI' \
  | paste - - \
  | sort | uniq -c | sort -rn | head -20
```

## Step 7: Optimize Cluster Agent Resources

```yaml
# Set limits for cattle-cluster-agent to avoid resource contention
# Apply this to each managed cluster
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cattle-cluster-agent
  namespace: cattle-system
spec:
  template:
    spec:
      containers:
        - name: agent
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

## Step 8: Configure Rancher Audit Logging

```yaml
# Enable audit logging with manageable verbosity
# rancher-audit-values.yaml
auditLog:
  level: 1  # 1=metadata only, 2=request body, 3=request+response body
  maxAge: 10
  maxBackups: 10
  maxSize: 100  # MB

# For performance, use level 1 in production
# Level 3 is very verbose and impacts performance
```

## Conclusion

Optimizing Rancher server performance requires attention to multiple layers: compute resources, etcd health, API server configuration, and cluster agent sizing. For deployments managing many clusters, proactive monitoring of Rancher's resource usage and API latency is essential for maintaining responsiveness. The key optimization levers are increasing heap size, tuning etcd compaction, adjusting reconciliation intervals, and ensuring Rancher pods are scheduled on nodes with adequate CPU and memory resources.
