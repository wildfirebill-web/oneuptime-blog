# How to Deploy Kubernetes Operators via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Operators, CRDs, DevOps

Description: Deploy and manage Kubernetes Operators that extend cluster functionality using Portainer's Kubernetes YAML interface.

## Introduction

Kubernetes Operators are controllers that extend Kubernetes functionality for specific applications. They use Custom Resource Definitions (CRDs) to manage complex applications like databases, monitoring stacks, and message queues. Portainer's YAML editor supports deploying Operators and their CRDs.

## Deploying the Cert-Manager Operator

```bash
# Deploy Cert-Manager via Portainer's Helm integration
# Kubernetes > Helm Charts > Search "cert-manager"
# Or install via kubectl manifest:

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Verify installation
kubectl get pods -n cert-manager
```

## Deploy Prometheus Operator via Portainer

```bash
# Add the Prometheus community Helm repo in Portainer
# Kubernetes > Helm Repositories > Add
# Name: prometheus-community
# URL: https://prometheus-community.github.io/helm-charts

# Then deploy via Helm Charts
# Search: kube-prometheus-stack
# Configure values and install
```

## Deploying a Custom Operator via YAML

```yaml
# example-operator.yml - deploy via Portainer YAML editor
# Step 1: Deploy the CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.postgres.example.com
spec:
  group: postgres.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
              version:
                type: string
  scope: Namespaced
  names:
    plural: postgresclusters
    singular: postgrescluster
    kind: PostgresCluster
---
# Step 2: Deploy the Operator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-operator
  namespace: operators
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-operator
  template:
    metadata:
      labels:
        app: postgres-operator
    spec:
      serviceAccountName: postgres-operator-sa
      containers:
      - name: operator
        image: postgres-operator:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

## Using Custom Resources After Operator Deployment

```yaml
# Create a managed PostgreSQL cluster via CRD
apiVersion: postgres.example.com/v1
kind: PostgresCluster
metadata:
  name: production-db
  namespace: production
spec:
  replicas: 3
  version: "15"
  storage:
    size: 50Gi
    storageClass: fast-ssd
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
```

## Popular Operators to Deploy via Portainer

| Operator | Purpose | Helm Chart |
|----------|---------|------------|
| cert-manager | TLS certificates | cert-manager/cert-manager |
| Prometheus Operator | Monitoring | prometheus-community/kube-prometheus-stack |
| PostgreSQL Operator | Managed PostgreSQL | zalionis/postgres-operator |
| MinIO Operator | Object storage | minio/operator |
| Strimzi Kafka Operator | Apache Kafka | strimzi/strimzi-kafka-operator |
| Vault Operator | Secrets management | hashicorp/vault |

## Monitoring Operators in Portainer

Operators appear as standard Deployments in Portainer:
- **Kubernetes > Applications** — shows Operator pods
- **Kubernetes > Namespaces** — shows Operator namespace resources
- Custom resources appear under **Kubernetes > Advanced** for raw YAML access

```bash
# Check Operator status
kubectl get pods -n operators
kubectl get crd | grep example.com

# View Custom Resource instances
kubectl get postgresclusters -A
```

## Conclusion

Kubernetes Operators deployed via Portainer extend cluster capabilities with managed services for databases, monitoring, and more. Portainer's YAML editor and Helm integration make deploying Operators straightforward, while the workload views provide operational visibility into Operator pods and their managed resources. Custom resources created through Operators are accessible via Portainer's Kubernetes interface.
