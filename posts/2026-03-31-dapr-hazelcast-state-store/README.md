# How to Configure Dapr with Hazelcast State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Hazelcast, State Store, Distributed Cache, Microservice

Description: Learn how to configure Dapr with Hazelcast as a state store, using Hazelcast's in-memory data grid for high-performance distributed state in microservices.

---

## Overview

Hazelcast is an in-memory data grid (IMDG) that provides distributed data structures, including maps, queues, and sets, with automatic partitioning and replication. When used as a Dapr state store, Hazelcast delivers low-latency reads and writes with automatic failover across cluster nodes.

## Prerequisites

- A running Hazelcast cluster (version 4.x or 5.x)
- Dapr CLI and runtime installed
- Docker or Kubernetes environment

## Setting Up Hazelcast

Start a Hazelcast cluster using Docker Compose:

```yaml
version: "3.8"
services:
  hazelcast-1:
    image: hazelcast/hazelcast:5.3
    environment:
      - HZ_CLUSTERNAME=dapr-cluster
    ports:
      - "5701:5701"

  hazelcast-2:
    image: hazelcast/hazelcast:5.3
    environment:
      - HZ_CLUSTERNAME=dapr-cluster
    ports:
      - "5702:5701"
```

Start the cluster:

```bash
docker-compose up -d
```

Verify both members have joined the cluster by checking the logs:

```bash
docker-compose logs hazelcast-1 | grep "Members"
```

## Configuring the Dapr Component

Create the Dapr state store component manifest:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: hazelcast-statestore
  namespace: default
spec:
  type: state.hazelcast
  version: v1
  metadata:
  - name: hazelcastServers
    value: "localhost:5701,localhost:5702"
  - name: hazelcastMap
    value: "dapr-state"
```

Apply the component:

```bash
kubectl apply -f hazelcast-statestore.yaml
```

For self-hosted mode, copy the file to your Dapr components directory:

```bash
cp hazelcast-statestore.yaml ~/.dapr/components/
```

## Using the Hazelcast State Store

Save and retrieve state using the Dapr SDK:

```javascript
import { DaprClient } from "@dapr/dapr";

const client = new DaprClient();

// Save user session
await client.state.save("hazelcast-statestore", [
  {
    key: "user-session-789",
    value: {
      userId: 789,
      permissions: ["read", "write"],
      expiresAt: new Date(Date.now() + 3600 * 1000).toISOString()
    }
  }
]);

// Read session
const session = await client.state.get("hazelcast-statestore", "user-session-789");
console.log("Session data:", session);

// Delete on logout
await client.state.delete("hazelcast-statestore", "user-session-789");
```

## Testing Failover

Hazelcast automatically redistributes data when a node leaves the cluster. Test this by stopping one node:

```bash
docker-compose stop hazelcast-1
```

State operations should continue to succeed against `hazelcast-2`. Restart the node and verify data is still accessible:

```bash
docker-compose start hazelcast-1
curl http://localhost:3500/v1.0/state/hazelcast-statestore/user-session-789
```

## Summary

Dapr's Hazelcast state store component leverages Hazelcast's in-memory data grid for distributed, low-latency state management. Its automatic data partitioning and built-in failover make it an excellent choice for microservices that require high availability and horizontal scalability without the overhead of disk-based persistence.
