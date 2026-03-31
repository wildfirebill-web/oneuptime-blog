# How to Configure MySQL Headless Service on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Kubernetes, Service, StatefulSet, Network

Description: Configure a Kubernetes headless service for MySQL StatefulSets to enable stable DNS-based pod addressing and support replication and cluster topologies.

---

A headless service in Kubernetes is a Service with `clusterIP: None`. Instead of providing a single virtual IP for load balancing, it creates individual DNS A records for each pod. This is essential for MySQL StatefulSets where each pod needs a stable, predictable hostname for replication and cluster communication.

## Why MySQL Needs a Headless Service

When MySQL runs as a StatefulSet, each pod needs a stable network identity. A regular ClusterIP service would route connections to any pod, which breaks replication setups where writes must go to the primary and reads to replicas. The headless service assigns each pod a DNS name in the format `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.

For example, with a StatefulSet named `mysql` and headless service `mysql`, the pods get these DNS records:

```text
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

## Creating the Headless Service

Define the headless service with `clusterIP: None`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306
```

The `selector` must match the pod labels in the StatefulSet. The `clusterIP: None` field is what makes it headless.

## Referencing the Headless Service in a StatefulSet

The StatefulSet references the headless service via the `serviceName` field:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Adding a Read/Write Service Alongside the Headless Service

In practice, you also create a regular ClusterIP service pointing to the primary pod, using a label that only the primary pod carries:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-primary
spec:
  selector:
    app: mysql
    role: primary
  ports:
  - port: 3306
    targetPort: 3306
```

## Verifying DNS Resolution

After deploying, verify the headless service creates per-pod DNS entries:

```bash
kubectl get svc mysql
kubectl get pods -l app=mysql

# Test DNS resolution from a debug pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup mysql-0.mysql.default.svc.cluster.local
```

The nslookup should return the pod IP directly rather than a virtual IP.

## Checking Connectivity

Connect to a specific MySQL pod by its DNS name:

```bash
kubectl run -it --rm mysql-client \
  --image=mysql:8.0 \
  --restart=Never \
  -- mysql -h mysql-0.mysql.default.svc.cluster.local -uroot -p
```

## Summary

The headless service with `clusterIP: None` is a fundamental building block for MySQL on Kubernetes. It gives each StatefulSet pod a predictable, stable DNS hostname that persists across pod restarts. This DNS-based addressing makes it possible to configure MySQL replication targets, InnoDB Cluster members, and routing rules with stable identifiers rather than ephemeral IP addresses.
