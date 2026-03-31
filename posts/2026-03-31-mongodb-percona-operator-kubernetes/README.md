# How to Use the Percona Operator for MongoDB on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kubernetes, Percona, Operator, Replica Set

Description: Learn how to deploy and manage MongoDB clusters on Kubernetes using the Percona Operator, including sharded clusters, backups, and monitoring integration.

---

## Overview

The Percona Operator for MongoDB (PSMDB Operator) is an open-source Kubernetes operator that manages Percona Server for MongoDB - a drop-in MongoDB-compatible distribution. It supports replica sets, sharded clusters, automated backups, and monitoring integration out of the box.

## Installing the Percona Operator

```bash
# Deploy the CRDs and operator
kubectl apply -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v1.15.0/deploy/bundle.yaml

# Verify
kubectl get pods -n default -l app.kubernetes.io/name=percona-server-mongodb-operator
```

Or use Helm:

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update
helm install psmdb-operator percona/psmdb-operator --namespace psmdb --create-namespace
```

## Deploying a Replica Set Cluster

Create a `PerconaServerMongoDB` custom resource:

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: my-cluster
  namespace: psmdb
spec:
  image: percona/percona-server-mongodb:7.0.4-1
  replsets:
    - name: rs0
      size: 3
      storage:
        engine: wiredTiger
      volumeSpec:
        persistentVolumeClaim:
          resources:
            requests:
              storage: 20Gi
  secrets:
    users: psmdb-secrets
```

Create the required secrets:

```bash
kubectl create secret generic psmdb-secrets \
  --from-literal=MONGODB_BACKUP_USER=backup \
  --from-literal=MONGODB_BACKUP_PASSWORD=backuppass \
  --from-literal=MONGODB_DATABASE_ADMIN_USER=admin \
  --from-literal=MONGODB_DATABASE_ADMIN_PASSWORD=adminpass \
  --from-literal=MONGODB_CLUSTER_ADMIN_USER=clusterAdmin \
  --from-literal=MONGODB_CLUSTER_ADMIN_PASSWORD=clusterAdminPass \
  --from-literal=MONGODB_CLUSTER_MONITOR_USER=clusterMonitor \
  --from-literal=MONGODB_CLUSTER_MONITOR_PASSWORD=monitorPass \
  -n psmdb
```

Apply:

```bash
kubectl apply -f psmdb-cluster.yaml
```

## Configuring Automated Backups

```yaml
spec:
  backup:
    enabled: true
    image: percona/percona-backup-mongodb:2.3.1
    serviceAccountName: percona-server-mongodb-operator
    storages:
      s3-backup:
        type: s3
        s3:
          bucket: my-mongodb-backups
          region: us-east-1
          credentialsSecret: s3-credentials
    tasks:
      - name: daily-backup
        enabled: true
        schedule: "0 2 * * *"
        keep: 7
        storageName: s3-backup
```

## Enabling Monitoring with PMM

```yaml
spec:
  pmm:
    enabled: true
    image: percona/pmm-client:2
    serverHost: pmm-server.monitoring.svc.cluster.local
    serverUser: admin
    pmmserverpassword: pmmpass
```

## Checking Cluster Status

```bash
kubectl get psmdb -n psmdb
kubectl describe psmdb my-cluster -n psmdb

# Connect to the cluster
kubectl exec -it my-cluster-rs0-0 -n psmdb -- mongosh \
  "mongodb://admin:adminpass@localhost:27017/admin"
```

## Summary

The Percona Operator for MongoDB provides a production-grade way to run MongoDB-compatible clusters on Kubernetes with built-in support for sharding, automated backups to S3, and integration with Percona Monitoring and Management (PMM). It is a strong choice for teams wanting the MongoDB API with enhanced enterprise-grade tooling and a fully open-source stack.
