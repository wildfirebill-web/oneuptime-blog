# How to Deploy DaemonSets in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, DaemonSet, DevOps

Description: Learn how to deploy and manage Kubernetes DaemonSets in Portainer for running workloads on every cluster node.

## Introduction

DaemonSets ensure that a pod runs on every node (or a subset of nodes) in a Kubernetes cluster. They are ideal for cluster-level services like log collectors, monitoring agents, network plugins, and security scanners. Portainer allows deploying DaemonSets via YAML manifests and viewing them in the Applications list. This guide covers DaemonSet deployment and management.

## Prerequisites

- Portainer with Kubernetes environment
- Admin access to the cluster
- Understanding of when to use DaemonSets

## Use Cases for DaemonSets

- **Logging**: Fluentd, Filebeat, Promtail
- **Monitoring**: Node Exporter, Datadog agent, New Relic agent
- **Storage**: Longhorn storage daemon, GlusterFS
- **Networking**: Calico, Flannel, Cilium network agents
- **Security**: Falco, Wazuh agents
- **GPU drivers**: NVIDIA device plugin

## Step 1: Deploy a DaemonSet via Portainer YAML

1. Select your Kubernetes environment in Portainer
2. Navigate to **Applications → Add application**
3. Select **YAML** editor
4. Enter the DaemonSet manifest:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
    component: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter

  # How DaemonSet updates are handled
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Update one node at a time

  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Allow running on control-plane nodes too
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

      # Run as host network for node-level metrics
      hostNetwork: true
      hostPID: true

      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          args:
            - "--path.procfs=/host/proc"
            - "--path.sysfs=/host/sys"
            - "--path.rootfs=/host/root"
            - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|run|host|etc)($$|/)"
          ports:
            - containerPort: 9100
              name: metrics
              hostPort: 9100
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
          securityContext:
            runAsUser: 0    # Needs root for node metrics

      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

5. Click **Deploy**

## Step 2: Deploy a Log Collector DaemonSet

Fluentd for log aggregation on all nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: elasticsearch.logging.svc.cluster.local
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

## Step 3: Deploy DaemonSet to Subset of Nodes

Use node selector or affinity to run only on specific nodes:

```yaml
spec:
  template:
    spec:
      # Only run on nodes labeled with "gpu=true"
      nodeSelector:
        accelerator: nvidia-gpu

      # Or use node affinity for more complex rules
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In
                    values:
                      - high-memory
                      - database
```

## Step 4: View DaemonSets in Portainer

1. Go to **Applications** in the Kubernetes environment
2. DaemonSets appear in the applications list with type "DaemonSet"
3. The application shows the desired/ready pod count:

```text
NAME            TYPE         PODS     NAMESPACE    AGE
node-exporter   DaemonSet    5/5      monitoring   2d
fluentd         DaemonSet    5/5      logging      5d
```

Click on a DaemonSet to see individual pod placement per node.

## Step 5: Check DaemonSet Status

```bash
# View DaemonSet details

kubectl get daemonset -n monitoring

# Output:
# NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# node-exporter  5         5         5       5            5           <none>

# Describe for issues
kubectl describe daemonset node-exporter -n monitoring

# Check which nodes have pods
kubectl get pods -n monitoring -l app=node-exporter -o wide
```

## Step 6: Update a DaemonSet

To update a DaemonSet (e.g., new image version):

1. Edit the DaemonSet in Portainer YAML
2. Update the image tag
3. Apply the update

```yaml
containers:
  - name: node-exporter
    image: prom/node-exporter:v1.8.0    # Updated version
```

With `RollingUpdate` strategy, Portainer/Kubernetes updates one node at a time.

```bash
# Monitor rolling update
kubectl rollout status daemonset/node-exporter -n monitoring
```

## Step 7: DaemonSet on New Nodes

When a new node joins the cluster, Kubernetes automatically creates the DaemonSet pod on it. This is one of the key advantages of DaemonSets over Deployments for cluster-level services.

## Conclusion

DaemonSets are essential for cluster-wide services that need to run on every node. Portainer supports deploying and managing DaemonSets through YAML manifests, and they appear alongside Deployments and StatefulSets in the Applications list. Key design considerations include tolerations to run on control-plane nodes and node selectors to target specific node types.
