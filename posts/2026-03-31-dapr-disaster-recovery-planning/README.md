# How to Plan Disaster Recovery for Dapr Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Disaster Recovery, Kubernetes, Resilience, High Availability

Description: A step-by-step guide to planning disaster recovery for Dapr applications, covering RTO/RPO targets, component replication, and recovery runbooks.

---

## Disaster Recovery Planning for Dapr

Dapr applications depend on external components - state stores, pub/sub brokers, and bindings. A disaster recovery (DR) plan must account for the failure of any of these dependencies, not just the application pods themselves. Planning for DR means defining recovery objectives, identifying failure domains, and preparing tested runbooks.

## Defining Recovery Objectives

Establish RPO (Recovery Point Objective) and RTO (Recovery Time Objective) for each service:

```yaml
# dr-objectives.yaml - store in your infrastructure repo
services:
  payment-service:
    rpo: "5m"
    rto: "15m"
    tier: "critical"
    components:
    - name: payments-statestore
      type: state.redis
      replication: active-passive
    - name: payments-pubsub
      type: pubsub.kafka
      replication: multi-az

  notification-service:
    rpo: "1h"
    rto: "2h"
    tier: "standard"
    components:
    - name: notifications-pubsub
      type: pubsub.redis
      replication: none
```

## Component Failure Domain Analysis

Map each Dapr component to its failure domain:

```bash
#!/bin/bash
# analyze-failure-domains.sh
echo "=== Dapr Component Failure Domain Analysis ==="

kubectl get components --all-namespaces -o json | jq '
  .items[] | {
    component: .metadata.name,
    namespace: .metadata.namespace,
    type: .spec.type,
    backend: (
      .spec.metadata // [] |
      map(select(.name | test("host|url|broker|endpoint"; "i"))) |
      first | .value // "unknown"
    )
  }
'
```

## DR Architecture Diagram

Design your DR topology with primary and standby regions:

```text
Primary Region (us-east-1)
  Kubernetes Cluster A
  - Dapr runtime
  - Services
  - Redis Primary (state)
  - Kafka Primary (pub/sub)
          |
          | Async replication
          v
DR Region (us-west-2)
  Kubernetes Cluster B
  - Dapr runtime (standby)
  - Services (scaled to 0)
  - Redis Replica (state)
  - Kafka Mirror (pub/sub)
```

## Implementing the DR Runbook

Document the recovery steps as executable scripts:

```bash
#!/bin/bash
# dr-runbook-failover.sh

NAMESPACE="production"
DR_CLUSTER_CONTEXT="k8s-dr-cluster"
PRIMARY_CONTEXT="k8s-primary-cluster"

echo "[Step 1] Verify DR cluster is healthy"
kubectl --context="$DR_CLUSTER_CONTEXT" get nodes

echo "[Step 2] Check component health in DR cluster"
kubectl --context="$DR_CLUSTER_CONTEXT" get components -n "$NAMESPACE"

echo "[Step 3] Scale up services in DR cluster"
kubectl --context="$DR_CLUSTER_CONTEXT" scale deployment --all \
  --replicas=3 -n "$NAMESPACE"

echo "[Step 4] Update DNS to point to DR cluster"
# Update Route53 or equivalent
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch file://dr-dns-failover.json

echo "[Step 5] Verify Dapr health in DR cluster"
kubectl --context="$DR_CLUSTER_CONTEXT" \
  get pods -n dapr-system

echo "[Step 6] Test service invocation"
curl -f "http://dr-load-balancer/health" && echo "DR services healthy"
```

## Configuring Dapr for DR Readiness

Pre-deploy all Dapr components in the DR cluster so they are ready when needed:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: production
  annotations:
    dr-region: "us-west-2"
    dr-priority: "1"
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-dr.us-west-2.internal:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
```

## Summary

Disaster recovery planning for Dapr applications requires defining RPO and RTO targets per service tier, mapping all Dapr components to their failure domains, and pre-deploying component configurations in the DR cluster. Document recovery steps as executable runbooks and test them on a quarterly schedule. Keep DR cluster services scaled to zero to minimize cost while maintaining readiness, and automate DNS failover as part of the recovery procedure.
