# How to Deploy MongoDB on Kubernetes with StatefulSets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kubernetes, StatefulSet, Replica Set, Deployment

Description: Learn how to deploy a MongoDB replica set on Kubernetes using StatefulSets, persistent volume claims, and headless services for stable network identities.

---

## Overview

StatefulSets are the standard Kubernetes workload for databases like MongoDB. They provide stable pod names, stable network identities, and ordered deployment/scaling - all critical for running a MongoDB replica set.

## Creating a Headless Service

A headless service gives each pod a stable DNS name:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: default
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

Each pod gets a DNS name like `mongodb-0.mongodb.default.svc.cluster.local`.

## Creating a Secret for MongoDB Credentials

```bash
kubectl create secret generic mongodb-secret \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=admin \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=securepassword
```

## Deploying the StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: "mongodb"
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
          command:
            - mongod
            - "--replSet"
            - "rs0"
            - "--bind_ip_all"
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

Apply the manifest:

```bash
kubectl apply -f mongodb-statefulset.yaml
```

## Initializing the Replica Set

Connect to `mongodb-0` and initialize the replica set:

```bash
kubectl exec -it mongodb-0 -- mongosh -u admin -p securepassword
```

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb.default.svc.cluster.local:27017" },
    { _id: 1, host: "mongodb-1.mongodb.default.svc.cluster.local:27017" },
    { _id: 2, host: "mongodb-2.mongodb.default.svc.cluster.local:27017" }
  ]
})
```

## Verifying the Deployment

```bash
kubectl get pods -l app=mongodb
kubectl get pvc
```

```javascript
rs.status()  // All members should show state PRIMARY or SECONDARY
```

## Exposing MongoDB to Applications

Create a ClusterIP service for application access:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-svc
spec:
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
```

Application connection string:

```bash
mongodb://admin:securepassword@mongodb-svc.default.svc.cluster.local:27017/mydb?replicaSet=rs0&authSource=admin
```

## Summary

Deploying MongoDB on Kubernetes with StatefulSets gives each pod a stable DNS identity, which is required for replica set initialization. Use `volumeClaimTemplates` to provision a PersistentVolumeClaim for each pod, and a headless service to expose the stable pod DNS names. After deployment, initialize the replica set manually by connecting to `mongodb-0`.
