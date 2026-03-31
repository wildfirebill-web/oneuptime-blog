# How to Use Dapr for Hybrid Cloud Deployments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Hybrid Cloud, On-Premises, Kubernetes, Multi-Environment

Description: Configure Dapr to run services across on-premises and cloud Kubernetes clusters, using component abstraction to maintain consistent APIs regardless of where services run.

---

## Hybrid Cloud with Dapr

A hybrid cloud deployment runs workloads across on-premises infrastructure and one or more public clouds. Dapr's component abstraction makes hybrid deployments more manageable: services use the same Dapr APIs regardless of where they run, and only the component YAML changes per environment.

## Architecture Overview

```yaml
hybrid_architecture:
  on_premises:
    cluster: "k8s-datacenter-1"
    services: ["order-processor", "inventory-service"]
    components:
      state: "state.redis (on-prem Redis)"
      pubsub: "pubsub.kafka (on-prem Kafka)"
      secrets: "secretstores.hashicorp.vault"

  cloud:
    provider: "AWS"
    cluster: "eks-us-east-1"
    services: ["customer-api", "notification-service"]
    components:
      state: "state.aws.dynamodb"
      pubsub: "pubsub.aws.snssqs"
      secrets: "secretstores.aws.secretmanager"
```

Services in both environments call each other using the same Dapr service invocation API.

## Component Configuration Per Environment

The same service code works in both environments. Only component YAML differs:

```yaml
# On-premises: components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis.datacenter.internal:6379
```

```yaml
# Cloud: components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.aws.dynamodb
  version: v1
  metadata:
    - name: table
      value: my-state-table
    - name: region
      value: us-east-1
```

## Cross-Environment Service Invocation

Use Dapr name resolution to invoke services across environments. Configure a custom name resolver or use the Consul resolver for hybrid networks:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: name-resolver
spec:
  type: nameresolution.consul
  version: v1
  metadata:
    - name: client.address
      value: consul.datacenter.internal:8500
    - name: tags
      value: dapr
    - name: healthCheckInterval
      value: "10"
```

Register services with Consul in both environments and Dapr will resolve them cross-environment.

## Cross-Environment Pub/Sub with Kafka

Use a Kafka cluster accessible from both environments as the shared message bus:

```yaml
# Same Kafka component deployed to both clusters
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cross-env-pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: kafka.shared.example.com:9092
    - name: consumerGroup
      value: on-prem-consumers      # Different per environment
    - name: authRequired
      value: "true"
    - name: saslUsername
      secretKeyRef:
        name: kafka-secret
        key: username
```

## Network Connectivity

Establish connectivity between environments using:

```bash
# Option 1: VPN tunnel (IPSec or WireGuard)
# Option 2: Dedicated interconnect (AWS Direct Connect, Azure ExpressRoute)
# Option 3: Cloudflare Tunnel for lightweight connectivity

# Verify connectivity between on-prem and cloud Dapr services
kubectl exec -it order-processor-pod -c daprd -- \
  wget -qO- http://customer-api.eks-us-east-1.internal:80/healthz
```

## Secrets Management Across Environments

Use a centralized Vault instance accessible from both environments:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-secrets
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    - name: vaultAddr
      value: https://vault.datacenter.internal:8200
    - name: skipVerify
      value: "false"
    - name: tlsCACert
      secretKeyRef:
        name: vault-ca
        key: ca.crt
```

Both on-premises and cloud services reference the same Vault instance.

## Observability Across Environments

Configure a central Zipkin or Grafana instance to receive traces from both environments:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing-config
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: https://observability.example.com/api/v2/spans
```

## Summary

Dapr enables hybrid cloud deployments by keeping application code identical across environments while varying only the component YAML files. Services use the same state, pub/sub, and secret APIs regardless of whether they run on-premises or in the cloud. Cross-environment connectivity via VPN or dedicated interconnects, shared Kafka for pub/sub, and a centralized Vault instance for secrets management complete the hybrid architecture.
