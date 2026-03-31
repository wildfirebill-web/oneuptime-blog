# How to Use Dapr for Multi-Cloud Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Multi-Cloud, AWS, Azure, GCP, Portability

Description: Use Dapr's component abstraction to deploy the same services across AWS, Azure, and GCP by swapping cloud-specific component implementations without code changes.

---

## Multi-Cloud Portability with Dapr

One of Dapr's key advantages is cloud portability. The same service code that reads state from AWS DynamoDB can run on Azure using Cosmos DB or on GCP using Firestore - with only a YAML file change.

## Component Equivalents Across Clouds

| Building Block | AWS | Azure | GCP |
|---|---|---|---|
| State Store | DynamoDB | Cosmos DB | Firestore |
| Pub/Sub | SNS/SQS | Service Bus | Pub/Sub |
| Secret Store | Secrets Manager | Key Vault | Secret Manager |
| Bindings | S3, Lambda | Blob Storage, Functions | GCS, Cloud Functions |

## Structuring Multi-Cloud Components

Use Kustomize overlays to manage cloud-specific configurations:

```bash
components/
├── base/
│   └── resiliency.yaml           # Cloud-agnostic
├── overlays/
│   ├── aws/
│   │   ├── statestore.yaml
│   │   ├── pubsub.yaml
│   │   └── secrets.yaml
│   ├── azure/
│   │   ├── statestore.yaml
│   │   ├── pubsub.yaml
│   │   └── secrets.yaml
│   └── gcp/
│       ├── statestore.yaml
│       ├── pubsub.yaml
│       └── secrets.yaml
```

Deploy to target cloud:

```bash
kubectl apply -k components/overlays/aws/ -n production
# or
kubectl apply -k components/overlays/azure/ -n production
```

## AWS Component Configuration

```yaml
# AWS DynamoDB state store
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.aws.dynamodb
  version: v1
  metadata:
    - name: table
      value: dapr-state
    - name: region
      value: us-east-1
    - name: accessKey
      secretKeyRef:
        name: aws-secret
        key: accessKeyId
    - name: secretKey
      secretKeyRef:
        name: aws-secret
        key: secretAccessKey
```

## Azure Component Configuration

```yaml
# Azure Cosmos DB state store
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
    - name: url
      value: https://myaccount.documents.azure.com:443/
    - name: masterKey
      secretKeyRef:
        name: cosmos-secret
        key: masterKey
    - name: database
      value: dapr-db
    - name: collection
      value: dapr-state
```

## GCP Component Configuration

```yaml
# GCP Firestore state store
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.gcp.firestore
  version: v1
  metadata:
    - name: type
      value: serviceaccount
    - name: project_id
      value: my-gcp-project
    - name: private_key_id
      secretKeyRef:
        name: gcp-secret
        key: private_key_id
    - name: collection
      value: dapr-state
```

## Multi-Cloud Pub/Sub

Use a cloud-agnostic Kafka or Redis as the cross-cloud message bus if you need cross-cloud pub/sub:

```yaml
# Cross-cloud pub/sub using Confluent Cloud (managed Kafka)
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: pkc-xxx.us-east-1.aws.confluent.cloud:9092
    - name: authRequired
      value: "true"
    - name: saslMechanism
      value: PLAIN
    - name: saslUsername
      secretKeyRef:
        name: confluent-secret
        key: api-key
    - name: saslPassword
      secretKeyRef:
        name: confluent-secret
        key: api-secret
```

## CI/CD for Multi-Cloud Deployments

Use a pipeline that deploys to multiple clouds sequentially:

```yaml
# GitHub Actions multi-cloud deploy
jobs:
  deploy-aws:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
      - run: kubectl apply -k components/overlays/aws/

  deploy-azure:
    runs-on: ubuntu-latest
    needs: deploy-aws
    steps:
      - uses: azure/login@v2
      - run: kubectl apply -k components/overlays/azure/
```

## Testing Multi-Cloud Consistency

Run the same integration test suite against each cloud:

```bash
#!/bin/bash
for cloud in aws azure gcp; do
  echo "Testing on $cloud..."
  export KUBECONFIG="kubeconfig-${cloud}"
  go test ./tests/integration/... \
    -tags=integration \
    -count=1 \
    -timeout=10m
  echo "$cloud: PASS"
done
```

## Summary

Dapr enables true multi-cloud portability by abstracting cloud-specific state stores, pub/sub brokers, and secret managers behind a uniform API. Using Kustomize overlays or Helm values files to manage cloud-specific component configurations - while keeping application code unchanged - allows the same services to run on AWS, Azure, and GCP. A managed Kafka cluster spanning clouds provides cross-cloud pub/sub when services need to communicate across cloud boundaries.
