# How to Optimize Rancher Server Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Performance, Optimization, Server Tuning, etcd, Kubernetes

Description: Optimize Rancher Server performance by tuning resource allocations, etcd configuration, database settings, and caching to handle large numbers of clusters and resources.

## Introduction

Rancher Server performance degrades as the number of managed clusters and Kubernetes resources grows. Common symptoms include slow UI response, delayed event processing, and high CPU usage on the Rancher Server pods. This guide covers key optimization areas.

## Step 1: Increase Rancher Server Resources

Rancher Server's resource requests are often too low for large deployments:

```yaml
# rancher-values.yaml (helm upgrade)

resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"

# Scale to multiple replicas for HA
replicas: 3
```

```bash
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --values rancher-values.yaml
```

## Step 2: Tune Rancher Environment Variables

Key Rancher tuning parameters are set via environment variables:

```yaml
# rancher-values.yaml additions
extraEnv:
  - name: CATTLE_AGENT_IMAGE
    value: "rancher/rancher-agent:v2.9.0"
  - name: CATTLE_CLUSTER_AGENT_DEFAULT_AFFINITY
    value: "prefer"
  # Reduce the frequency of Rancher's reconciliation loops
  - name: CATTLE_RESYNC_DEFAULT
    value: "30"    # Default is 15 seconds; increase to reduce CPU
  # Limit concurrent cluster reconciliations
  - name: CATTLE_WORKER_COUNT
    value: "50"    # Default 50; reduce if CPU is saturated
```

## Step 3: Optimize the Local Cluster (etcd)

The local Rancher cluster's etcd performance is critical. Defragment regularly:

```bash
# Defragment etcd on all etcd nodes
kubectl exec -n kube-system etcd-rancher-node-1 -- \
  etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/ssl/kube-ca.pem \
  --cert=/etc/kubernetes/ssl/kube-node.pem \
  --key=/etc/kubernetes/ssl/kube-node-key.pem
```

## Step 4: Configure etcd Quota

Increase the etcd database size if you hit quota errors:

```bash
# Current etcd DB size
etcdctl --endpoints=https://127.0.0.1:2379 endpoint status \
  --write-out=table

# Increase quota to 8GB (default 2GB)
# Set via RKE2 cluster config
cat > /etc/rancher/rke2/config.yaml << 'EOF'
etcd-arg:
  - "quota-backend-bytes=8589934592"
EOF
```

## Step 5: Reduce Audit Log Impact

Audit logging at high verbosity levels consumes significant CPU:

```yaml
# Reduce audit log level in cluster config
auditLog:
  level: 1    # Level 1 only logs request metadata (lowest impact)
  maxAge: 7   # Keep 7 days only
  maxSize: 100  # 100MB max file size
```

## Step 6: Database Tuning (External PostgreSQL)

For large deployments, migrate from SQLite to PostgreSQL:

```yaml
# rancher-values.yaml - external database
extraEnv:
  - name: CATTLE_DB_CATTLE_MYSQL_HOST
    value: "postgres.databases.svc.cluster.local"
  - name: CATTLE_DB_CATTLE_MYSQL_PORT
    value: "5432"
  - name: CATTLE_DB_CATTLE_MYSQL_NAME
    value: "rancher"
```

## Step 7: Monitor Rancher Server Performance

```bash
# Watch Rancher server resource usage
kubectl top pods -n cattle-system

# View leader election and reconciliation logs
kubectl logs -n cattle-system rancher-xxxxx | grep -E "requeue|enqueue" | head -50
```

## Conclusion

Rancher Server performance at scale requires tuning in multiple areas: resource allocations, reconciliation frequencies, etcd health, and database backend. The most impactful change for large deployments (50+ clusters) is migrating to an external PostgreSQL database and increasing the Rancher Server pod memory to 8GB or more.
