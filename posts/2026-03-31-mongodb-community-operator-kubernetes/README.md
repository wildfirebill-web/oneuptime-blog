# How to Use the MongoDB Community Operator on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kubernetes, Operator, Community, Replica Set

Description: Learn how to install and use the MongoDB Community Operator on Kubernetes to deploy and manage MongoDB replica sets with automated lifecycle operations.

---

## Overview

The MongoDB Community Kubernetes Operator automates deploying and managing MongoDB replica sets on Kubernetes. It handles replica set initialization, TLS, user management, and rolling upgrades through Kubernetes custom resources.

## Installing the Operator with Helm

```bash
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update

helm install community-operator mongodb/community-operator \
  --namespace mongodb \
  --create-namespace
```

Verify the operator is running:

```bash
kubectl get pods -n mongodb
```

## Creating a MongoDB Replica Set

Create a `MongoDBCommunity` custom resource:

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: my-mongodb
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.4"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: appuser
      db: mydb
      passwordSecretRef:
        name: appuser-password
      roles:
        - name: readWrite
          db: mydb
      scramCredentialsSecretName: appuser-scram
  additionalMongodConfig:
    storage.wiredTiger.engineConfig.journalCompressor: zlib
```

Create the password secret first:

```bash
kubectl create secret generic appuser-password \
  --from-literal=password=securepassword \
  -n mongodb
```

Apply the resource:

```bash
kubectl apply -f mongodb-community.yaml
```

## Checking Deployment Status

```bash
kubectl get mongodbcommunity -n mongodb
kubectl describe mongodbcommunity my-mongodb -n mongodb
```

The operator will create a StatefulSet, headless service, and persistent volume claims automatically.

## Connecting to the Replica Set

Get the connection string from the operator-generated secret:

```bash
kubectl get secret my-mongodb-admin-appuser -n mongodb \
  -o jsonpath='{.data.connectionString\.standardSrv}' | base64 --decode
```

## Upgrading MongoDB Version

To upgrade from 6.0 to 7.0, update the `version` field in the custom resource:

```yaml
spec:
  version: "7.0.4"
```

```bash
kubectl apply -f mongodb-community.yaml
```

The operator performs a rolling upgrade, upgrading each replica set member one at a time.

## Scaling the Replica Set

Change `members` in the spec and apply:

```yaml
spec:
  members: 5
```

```bash
kubectl apply -f mongodb-community.yaml
```

## Enabling TLS

```yaml
spec:
  security:
    tls:
      enabled: true
      certificateKeySecretRef:
        name: mongodb-tls-cert
      caConfigMapRef:
        name: mongodb-ca
```

## Summary

The MongoDB Community Operator simplifies managing MongoDB on Kubernetes by handling replica set initialization, user creation, rolling upgrades, and scaling through declarative custom resources. Install it via Helm, create a `MongoDBCommunity` resource with your desired configuration, and the operator handles the rest. Upgrades and scaling are as simple as editing the spec and applying it.
