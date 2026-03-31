# How to Set Operator Image and Log Level in Rook Helm Chart

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Helm, Operator, Logging

Description: Configure the Rook-Ceph operator container image version and log verbosity level through Helm chart values for better control and debugging.

---

## Overview

Two of the most frequently adjusted settings in the Rook-Ceph operator Helm chart are the container image and log level. The image version determines which Rook release runs your cluster, and the log level controls verbosity for troubleshooting.

## Setting the Operator Image

By default, the chart uses the image matching the Helm chart version. To pin a specific Rook image, override in your values file:

```yaml
image:
  repository: rook/ceph
  tag: v1.13.2
  pullPolicy: IfNotPresent
```

To use an image from a private registry:

```yaml
image:
  repository: registry.example.com/rook/ceph
  tag: v1.13.2
  pullPolicy: Always

imagePullSecrets:
  - name: registry-credentials
```

Apply the change:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  -f rook-operator-values.yaml
```

Verify the new image is running:

```bash
kubectl get pod -n rook-ceph -l app=rook-ceph-operator \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

## Configuring Log Level

Rook supports four log levels. Set the appropriate level for your situation:

```yaml
logLevel: INFO
```

Available levels:

```text
DEBUG   - Very verbose; use only during active troubleshooting
INFO    - Standard operational messages (default)
WARNING - Only warnings and errors
ERROR   - Errors only
```

To enable debug logging during an incident:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --set logLevel=DEBUG
```

## Viewing Operator Logs

After changing the log level, tail operator logs to see the effect:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator -f
```

For a specific time window during debugging:

```bash
kubectl logs -n rook-ceph -l app=rook-ceph-operator \
  --since=10m | grep -i error
```

## Combining Image and Log Level Override

A minimal values file addressing both settings:

```yaml
image:
  repository: rook/ceph
  tag: v1.13.2

logLevel: WARNING
```

This is often sufficient for a production cluster where you want a pinned release and reduced log noise.

## Reverting to Default Log Level

After debugging, restore INFO level to avoid log storage overhead:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --set logLevel=INFO
```

## Summary

Pinning the operator image and adjusting the log level are simple but high-impact changes to the Rook Helm chart. Use a values file to keep image tags version-controlled, and toggle log verbosity dynamically with `helm upgrade --set` during incidents to capture diagnostic detail without permanently inflating log volume.
