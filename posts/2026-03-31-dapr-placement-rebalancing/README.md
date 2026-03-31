# How to Handle Dapr Placement Rebalancing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Placement, Rebalancing, Kubernetes

Description: Learn how to handle Dapr placement rebalancing events to minimize actor redistribution disruptions and keep your distributed actor system stable.

---

## What Is Placement Rebalancing?

Dapr's Placement service maintains a consistent hash ring that maps actor types to specific host instances. When nodes join or leave a cluster, the Placement service triggers a rebalancing event - redistributing actor ownership across the updated set of hosts. During rebalancing, in-flight actor calls may be temporarily blocked until the new table is disseminated.

## How Rebalancing Works

The Placement service uses a gossip protocol to detect membership changes. When a new Dapr sidecar connects or an existing one disconnects, the service recalculates the hash ring and pushes an updated placement table to all connected sidecars.

```bash
# Watch placement rebalancing events in logs
kubectl logs -n dapr-system -l app=dapr-placement-server --follow | grep -i "rebalance\|table"
```

## Configuring Dissemination Settings

You can tune how aggressively Dapr pushes placement updates using Helm values.

```yaml
dapr_placement:
  replicaCount: 3
  extraArgs:
    - --disseminate-in-sec=2
    - --max-api-level=10
    - --min-api-level=0
```

Apply the configuration:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_placement.replicaCount=3 \
  --reuse-values
```

## Graceful Shutdown During Rebalancing

Configure actor hosts to drain active actors before shutdown to reduce lost work during rebalancing:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: actor-service
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: actor-service
          env:
            - name: DAPR_GRACEFUL_SHUTDOWN_SECONDS
              value: "25"
```

## Detecting Rebalancing in Your Application

Use the Dapr health endpoint to detect when your sidecar is in a rebalancing state and pause outgoing calls if needed:

```javascript
const axios = require('axios');

async function waitForSidecarReady() {
  const maxRetries = 30;
  for (let i = 0; i < maxRetries; i++) {
    try {
      const res = await axios.get('http://localhost:3500/v1.0/healthz/outbound');
      if (res.status === 204) return true;
    } catch {
      await new Promise(r => setTimeout(r, 1000));
    }
  }
  throw new Error('Sidecar did not become ready after rebalancing');
}
```

## Monitoring Rebalancing Frequency

Use Prometheus to track how often rebalancing occurs:

```bash
# Query placement rebalancing metric
dapr_placement_host_count
dapr_placement_table_update_total
```

Set up an alert if rebalancing is too frequent, which may indicate unstable node membership.

## Summary

Dapr Placement rebalancing redistributes actors when cluster membership changes. By tuning dissemination settings, configuring graceful shutdown, and monitoring rebalancing frequency with Prometheus metrics, you can minimize disruptions and keep actor workloads stable during scaling events or node failures.
