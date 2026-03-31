# How to Use Dapr with Oracle Cloud Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Oracle, OCI, Cloud, Microservice

Description: Integrate Dapr with Oracle Cloud Infrastructure services including OCI Object Storage and OCI Streaming for cloud-native microservices on OCI.

---

## Overview

Oracle Cloud Infrastructure (OCI) provides a broad suite of cloud services. Dapr can integrate with OCI through HTTP output bindings, OCI Streaming (Kafka-compatible), and native OCI component support. This guide covers the practical patterns for deploying Dapr-based microservices on OCI.

## Prerequisites

- OCI account with required services enabled
- OCI CLI configured with API key authentication
- Dapr installed on OCI Container Engine for Kubernetes (OKE)
- A compartment and tenancy OCID

## Deploying Dapr on OKE

```bash
# Get kubeconfig for OKE cluster
oci ce cluster create-kubeconfig \
  --cluster-id ocid1.cluster.oc1.xxx \
  --file $HOME/.kube/config \
  --region us-ashburn-1 \
  --token-version 2.0.0

# Install Dapr via Helm
helm repo add dapr https://dapr.github.io/helm-charts/
helm upgrade --install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --set global.ha.enabled=true \
  --wait
```

## Integrating with OCI Object Storage

Use Dapr's HTTP binding to call OCI Object Storage APIs:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: oci-storage
spec:
  type: bindings.http
  version: v1
  metadata:
  - name: url
    value: "https://objectstorage.us-ashburn-1.oraclecloud.com/n/my-namespace/b/my-bucket/o"
```

Upload an object:

```bash
curl -X POST http://localhost:3500/v1.0/bindings/oci-storage \
  -H "Content-Type: application/json" \
  -d '{
    "data": "SGVsbG8gT0NJIQ==",
    "metadata": {
      "path": "/my-object.txt",
      "Content-Type": "text/plain"
    },
    "operation": "put"
  }'
```

## Integrating with OCI Streaming (Kafka-Compatible)

OCI Streaming is Kafka-compatible, so use Dapr's Kafka pub/sub component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: oci-streaming
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: "cell-1.streaming.us-ashburn-1.oci.oraclecloud.com:9092"
  - name: authType
    value: "password"
  - name: saslUsername
    value: "my-tenancy/oracleidentitycloudservice/user@example.com/ocid1.streampool.oc1.xxx"
  - name: saslPassword
    secretKeyRef:
      name: oci-streaming-token
      key: token
  - name: consumerGroup
    value: "my-consumer-group"
```

## Using OCI Vault for Secrets

Configure Dapr to use OCI Vault via HTTP binding:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: oci-vault
spec:
  type: bindings.http
  version: v1
  metadata:
  - name: url
    value: "https://vaults.us-ashburn-1.oci.oraclecloud.com/20180608"
```

Or store OCI secrets in Kubernetes and reference them from Dapr:

```bash
kubectl create secret generic oci-credentials \
  --from-file=private-key=~/.oci/oci_api_key.pem \
  --from-literal=fingerprint=aa:bb:cc:dd:ee \
  --from-literal=tenancyId=ocid1.tenancy.oc1.xxx \
  --from-literal=userId=ocid1.user.oc1.xxx
```

## Observability with OCI Logging

Configure Dapr to send JSON logs compatible with OCI Logging:

```yaml
annotations:
  dapr.io/log-as-json: "true"
  dapr.io/log-level: "info"
```

## Summary

Dapr integrates with Oracle Cloud Infrastructure through HTTP bindings for OCI Object Storage, Kafka-compatible pub/sub for OCI Streaming, and standard Kubernetes secret management for OCI Vault credentials. OKE provides a managed Kubernetes environment where Dapr operates alongside OCI-native services effectively.
