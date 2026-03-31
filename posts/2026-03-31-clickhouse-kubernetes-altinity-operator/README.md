# How to Use ClickHouse Operator (Altinity) on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Altinity Operator, Operator Pattern, Cluster Management, K8s

Description: Learn how to deploy and manage ClickHouse clusters on Kubernetes using the Altinity ClickHouse Operator with ClickHouseInstallation custom resources.

---

The Altinity ClickHouse Operator is the most widely used Kubernetes operator for ClickHouse. It introduces the `ClickHouseInstallation` (CHI) custom resource, making it easy to declaratively manage ClickHouse clusters - shards, replicas, configuration, and storage.

## Installing the Operator

```bash
kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator/clickhouse-operator-install-bundle.yaml
```

Verify the operator is running:

```bash
kubectl get pods -n kube-system | grep clickhouse-operator
```

## Creating a Single-Node ClickHouseInstallation

```yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: clickhouse-simple
  namespace: clickhouse
spec:
  configuration:
    clusters:
      - name: "simple"
        layout:
          shardsCount: 1
          replicasCount: 1
```

Apply it:

```bash
kubectl apply -f clickhouse-simple.yaml
```

## Creating a Multi-Shard Replicated Cluster

```yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: "ClickHouseInstallation"
metadata:
  name: clickhouse-prod
  namespace: clickhouse
spec:
  configuration:
    zookeeper:
      nodes:
        - host: zookeeper.zookeeper.svc.cluster.local
          port: 2181
    clusters:
      - name: "prod_cluster"
        layout:
          shardsCount: 2
          replicasCount: 2
  defaults:
    templates:
      podTemplate: pod-template
      dataVolumeClaimTemplate: data-volume

  templates:
    podTemplates:
      - name: pod-template
        spec:
          containers:
            - name: clickhouse
              image: clickhouse/clickhouse-server:24.3
              resources:
                requests:
                  memory: "4Gi"
                  cpu: "2"
                limits:
                  memory: "8Gi"
                  cpu: "4"

    volumeClaimTemplates:
      - name: data-volume
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 100Gi
          storageClassName: fast-ssd
```

## Checking Installation Status

```bash
kubectl get chi -n clickhouse
kubectl describe chi clickhouse-prod -n clickhouse
kubectl get pods -n clickhouse -l clickhouse.altinity.com/chi=clickhouse-prod
```

## Connecting to the Cluster

```bash
# Get the service name
kubectl get svc -n clickhouse

# Port-forward for local access
kubectl port-forward svc/clickhouse-prod 9000:9000 -n clickhouse

# Connect
clickhouse-client --host localhost --port 9000
```

## Updating the Cluster

Edit the CHI manifest and re-apply. The operator handles rolling updates:

```bash
kubectl apply -f clickhouse-prod.yaml
```

## Summary

The Altinity ClickHouse Operator simplifies Kubernetes deployments through the `ClickHouseInstallation` custom resource. Define shards, replicas, storage, and resources declaratively, and the operator handles pod management, service creation, configuration distribution, and rolling updates automatically.
