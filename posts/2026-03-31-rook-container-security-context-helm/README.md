# How to Configure Container Security Context in Rook Helm Chart

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Helm, Security, Container

Description: Set pod and container security contexts for the Rook-Ceph operator via Helm to comply with Pod Security Standards and cluster security policies.

---

## Overview

Kubernetes Pod Security Standards and organizational policies often require explicit security contexts on containers. The Rook-Ceph operator Helm chart exposes `podSecurityContext` and `containerSecurityContext` values for the operator pod, and per-daemon overrides for Ceph components.

## Operator Pod Security Context

Configure the security context at the pod level to control user/group IDs and filesystem group:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 2016
  runAsGroup: 2016
  fsGroup: 2016
```

## Operator Container Security Context

Set security capabilities and privilege settings on the operator container itself:

```yaml
containerSecurityContext:
  runAsNonRoot: true
  runAsUser: 2016
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: false
  allowPrivilegeEscalation: false
```

Note that Rook does not require a read-only root filesystem by default because the operator needs to write configuration files during initialization.

## Applying the Configuration

Include these sections in your operator values file:

```yaml
# rook-operator-security.yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 2016
  runAsGroup: 2016
  fsGroup: 2016

containerSecurityContext:
  runAsNonRoot: true
  runAsUser: 2016
  capabilities:
    drop:
      - ALL
  allowPrivilegeEscalation: false
```

Then apply:

```bash
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  -f rook-operator-security.yaml
```

## Restricted Pod Security Standards

For clusters enforcing the `restricted` Pod Security Standard, ensure the namespace has the correct label:

```bash
kubectl label namespace rook-ceph \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

With the `restricted` standard, the above security context configuration aligns with all requirements. Verify after upgrade:

```bash
kubectl get pod -n rook-ceph -l app=rook-ceph-operator \
  -o jsonpath='{.items[0].spec.securityContext}'
```

## OSD Security Context Considerations

OSDs often need elevated privileges for disk access. Override this in the `CephCluster` CR rather than the operator chart:

```yaml
spec:
  security:
    keyRotation:
      enabled: false
```

For OSD pods that need `privileged: true`, Rook handles this internally. Do not attempt to restrict OSD pod security contexts through the operator chart - configure it in the cluster CR.

## Summary

Configuring security contexts for the Rook-Ceph operator via Helm ensures compliance with Pod Security Standards while keeping the operator functional. Apply `runAsNonRoot`, drop all capabilities, and prevent privilege escalation on the operator container while being careful not to apply the same restrictions to OSD daemons that legitimately need elevated disk access.
