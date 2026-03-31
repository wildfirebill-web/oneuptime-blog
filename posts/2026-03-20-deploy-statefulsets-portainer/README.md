# How to Deploy StatefulSets with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, StatefulSet, Docker, Persistent Storage, DevOps

Description: Learn how to create and manage Kubernetes StatefulSets through Portainer for stateful applications like databases and message queues.

---

StatefulSets manage pods that require stable network identities, ordered deployment, and persistent storage - ideal for databases, message queues, and caching systems. Portainer provides a UI to deploy and manage StatefulSets.

---

## Create a StatefulSet via Portainer

In Portainer, navigate to the Kubernetes cluster → **Applications** → **Add application** → **Advanced mode** and paste:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_DB
              value: appdb
            - name: POSTGRES_USER
              value: admin
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## Create the Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: default
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

The headless service gives each pod a stable DNS name: `postgres-0.postgres-headless.default.svc.cluster.local`.

---

## Scale the StatefulSet

```bash
# Scale via kubectl

kubectl scale statefulset postgres --replicas=3

# In Portainer: Workloads → StatefulSets → Edit → change replicas
```

StatefulSets scale in order: `postgres-0`, `postgres-1`, `postgres-2`.

---

## View Pod Stable Identities

```bash
kubectl get pods -l app=postgres
# NAME         READY   STATUS    RESTARTS
# postgres-0   1/1     Running   0
# postgres-1   1/1     Running   0

# Each pod's PVC
kubectl get pvc -l app=postgres
```

---

## Summary

StatefulSets provide stable pod names (`<name>-0`, `<name>-1`), stable DNS via headless services, and persistent storage through `volumeClaimTemplates`. Deploy them via Portainer's YAML editor with a headless service for DNS. Ordered deployment ensures `postgres-0` is running before `postgres-1` starts. Use StatefulSets for databases, message queues, and any workload needing stable identity.
