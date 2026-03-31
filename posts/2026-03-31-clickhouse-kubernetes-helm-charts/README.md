# How to Use Helm Charts for ClickHouse on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Helm, Helm Chart, Deployment, K8s

Description: Learn how to deploy ClickHouse on Kubernetes using Helm charts from Bitnami and Altinity, with custom values for production-ready configurations.

---

Helm charts provide a templated, versioned way to deploy ClickHouse on Kubernetes. Both Bitnami and Altinity publish official ClickHouse Helm charts that handle the complexity of cluster topology, persistent storage, and configuration management.

## Adding the Helm Repositories

```bash
# Bitnami chart
helm repo add bitnami https://charts.bitnami.com/bitnami

# Altinity chart
helm repo add altinity https://docs.altinity.com/clickhouse-operator/

helm repo update
```

## Deploying with the Bitnami Chart

```bash
# View available values
helm show values bitnami/clickhouse > clickhouse-values.yaml

# Install with defaults
helm install my-clickhouse bitnami/clickhouse \
  --namespace clickhouse \
  --create-namespace
```

## Custom values.yaml for Bitnami Chart

```yaml
# values.yaml
shards: 2
replicaCount: 2

auth:
  username: default
  password: "securepassword"
  existingSecret: ""

persistence:
  enabled: true
  storageClass: "fast-ssd"
  size: 100Gi

resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"

zookeeper:
  enabled: true
  replicaCount: 3

service:
  type: ClusterIP
  ports:
    http: 8123
    tcp: 9000
```

Deploy with custom values:

```bash
helm install my-clickhouse bitnami/clickhouse \
  --namespace clickhouse \
  --create-namespace \
  -f values.yaml
```

## Upgrading

```bash
helm upgrade my-clickhouse bitnami/clickhouse \
  --namespace clickhouse \
  -f values.yaml
```

## Checking Deployment Status

```bash
helm status my-clickhouse -n clickhouse
kubectl get pods -n clickhouse
kubectl get pvc -n clickhouse
```

## Port Forwarding for Local Access

```bash
kubectl port-forward svc/my-clickhouse 8123:8123 9000:9000 -n clickhouse
clickhouse-client --host localhost --port 9000
```

## Rolling Back

```bash
helm rollback my-clickhouse 1 -n clickhouse
```

## Uninstalling

```bash
helm uninstall my-clickhouse -n clickhouse
# Note: PVCs are not deleted automatically - clean up manually
kubectl delete pvc -l app.kubernetes.io/instance=my-clickhouse -n clickhouse
```

## Summary

Helm charts provide the fastest path to deploying ClickHouse on Kubernetes. Use the Bitnami chart for general-purpose deployments and the Altinity chart if you need the ClickHouse Operator's advanced management capabilities. Customize `values.yaml` for your storage class, resource limits, and shard/replica topology, then manage upgrades and rollbacks with standard Helm commands.
