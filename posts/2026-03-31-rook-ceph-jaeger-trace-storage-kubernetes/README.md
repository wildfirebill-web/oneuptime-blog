# How to Set Up Ceph for Jaeger Trace Storage on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Jaeger, Kubernetes, Storage, Tracing, Observability

Description: Configure Jaeger to store distributed traces in Ceph-backed storage on Kubernetes using either Cassandra on Ceph RBD or Elasticsearch on Ceph block volumes.

---

## Overview

Jaeger supports multiple storage backends including Cassandra, Elasticsearch, and Badger. On Kubernetes, you can back these databases with Ceph RBD volumes managed by Rook, giving Jaeger durable, scalable trace storage.

## Option 1 - Jaeger with Badger on Ceph RBD

Badger is Jaeger's embedded key-value store. Back it with a Ceph RBD PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jaeger-badger-pvc
  namespace: tracing
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 20Gi
```

Deploy Jaeger all-in-one with the Badger backend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: tracing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.55
          env:
            - name: SPAN_STORAGE_TYPE
              value: badger
            - name: BADGER_EPHEMERAL
              value: "false"
            - name: BADGER_DIRECTORY_VALUE
              value: /badger/data
            - name: BADGER_DIRECTORY_KEY
              value: /badger/key
          ports:
            - containerPort: 16686
            - containerPort: 4317
          volumeMounts:
            - name: badger-data
              mountPath: /badger
      volumes:
        - name: badger-data
          persistentVolumeClaim:
            claimName: jaeger-badger-pvc
```

## Option 2 - Jaeger with Elasticsearch on Ceph

Deploy Elasticsearch with Ceph RBD storage via Helm:

```bash
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch \
  --namespace tracing \
  --set persistence.storageClass=ceph-rbd \
  --set volumeClaimTemplate.resources.requests.storage=50Gi \
  --set replicas=3
```

Then deploy the Jaeger operator and configure it to use Elasticsearch:

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-prod
  namespace: tracing
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch-master:9200
        index-prefix: jaeger
```

## Verify Trace Ingestion

Port-forward Jaeger UI and query interface:

```bash
kubectl -n tracing port-forward svc/jaeger-query 16686
```

Send a test trace:

```bash
curl -X POST http://localhost:14268/api/traces \
  -H "Content-Type: application/x-thrift" \
  --data-binary @sample-trace.thrift
```

Open `http://localhost:16686` to view traces.

## Monitor Storage Usage

```bash
kubectl -n tracing get pvc
ceph df detail | grep -E "jaeger|badger"
```

## Summary

Jaeger integrates naturally with Ceph RBD-backed storage on Kubernetes. Whether using the embedded Badger backend for smaller deployments or Elasticsearch for production scale, Ceph provides the durable block storage layer that keeps trace data safe across restarts and upgrades.
