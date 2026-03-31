# How to Use Dapr with Kubernetes DaemonSets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kubernetes, DaemonSet, Node Agent, Log Collection

Description: Deploy Dapr-enabled DaemonSets to run node-level agents on every Kubernetes node for tasks like log forwarding, metric collection, and security scanning.

---

## Overview

DaemonSets ensure one pod runs on every node in a cluster. Combined with Dapr, you can build node-level agents that leverage Dapr's pub/sub and state management APIs for forwarding logs, metrics, and events from individual nodes.

## Dapr-Enabled DaemonSet

Deploy a DaemonSet with Dapr sidecar injection:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-metrics-agent
  namespace: default
spec:
  selector:
    matchLabels:
      app: node-metrics-agent
  template:
    metadata:
      labels:
        app: node-metrics-agent
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "node-metrics-agent"
        dapr.io/app-port: "3000"
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: metrics-agent
        image: myregistry/metrics-agent:latest
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          privileged: true
        volumeMounts:
        - name: host-proc
          mountPath: /host/proc
          readOnly: true
      volumes:
      - name: host-proc
        hostPath:
          path: /proc
```

## Publishing Node Metrics via Dapr

The agent collects node metrics and publishes them via Dapr pub/sub:

```javascript
const { DaprClient } = require('@dapr/dapr');
const os = require('os');

const client = new DaprClient();
const nodeName = process.env.NODE_NAME;

async function collectAndPublish() {
  const metrics = {
    nodeName,
    cpuUsage: process.cpuUsage(),
    memFree: os.freemem(),
    memTotal: os.totalmem(),
    timestamp: Date.now()
  };

  await client.pubsub.publish('pubsub', 'node-metrics', metrics);
  console.log(`Published metrics for node: ${nodeName}`);
}

setInterval(collectAndPublish, 30000);
```

## Accessing Host Resources

DaemonSets often need access to host-level resources. Configure the pod security context accordingly:

```yaml
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: metrics-agent
    securityContext:
      privileged: true
    volumeMounts:
    - name: host-sys
      mountPath: /host/sys
      readOnly: true
    - name: host-run
      mountPath: /host/run
  volumes:
  - name: host-sys
    hostPath:
      path: /sys
  - name: host-run
    hostPath:
      path: /run
```

## Limiting DaemonSet to Specific Nodes

Use node selectors or affinity to run the DaemonSet only on worker nodes:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-role: worker
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
```

## Verifying Dapr Sidecar on DaemonSet Pods

Check that Dapr sidecar was injected on all node pods:

```bash
kubectl get pods -l app=node-metrics-agent -o wide
kubectl get pods -l app=node-metrics-agent -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
# Should show: node-metrics-agent-xxx  metrics-agent daprd
```

## Summary

DaemonSets with Dapr enable powerful node-level agents that leverage Dapr's distributed capabilities for pub/sub and state management. By running one Dapr-enabled pod per node, you can aggregate node-level metrics, logs, and events into centralized systems without managing complex agent coordination logic. Apply tolerations carefully to control which node types the DaemonSet runs on.
