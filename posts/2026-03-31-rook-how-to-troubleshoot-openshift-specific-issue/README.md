# How to Troubleshoot OpenShift-Specific Issues with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Openshift, Troubleshooting, Kubernetes, Scc

Description: Learn how to diagnose and resolve OpenShift-specific issues with Rook-Ceph, including SCC misconfigurations, SELinux conflicts, and OLM deployment issues.

---

## Overview

Deploying Rook on OpenShift introduces additional constraints compared to vanilla Kubernetes. OpenShift's Security Context Constraints (SCCs), SELinux enforcement, and the Operator Lifecycle Manager (OLM) deployment model require specific configurations. This guide covers the most common OpenShift-specific issues and their resolutions.

## Issue 1 - Pods Failing Due to SCC Violations

OpenShift's SCC system prevents pods from running with elevated privileges by default. Rook requires certain privileges for OSD pods.

Check for SCC-related errors:

```bash
oc describe pod -n rook-ceph rook-ceph-osd-0-xxx | grep -A5 "forbidden"
```

The error typically looks like:

```text
Error creating: pods "rook-ceph-osd-0-" is forbidden: unable to validate against
any security context constraint: [provider "anyuid": Forbidden...]
```

Resolution - ensure the Rook service accounts have the correct SCCs:

```bash
oc adm policy add-scc-to-user privileged -z rook-ceph-system -n rook-ceph
oc adm policy add-scc-to-user privileged -z rook-ceph-osd -n rook-ceph
oc adm policy add-scc-to-user privileged -z rook-ceph-mgr -n rook-ceph
oc adm policy add-scc-to-user privileged -z rook-ceph-default -n rook-ceph
```

## Issue 2 - SELinux Blocking OSD Disk Access

OpenShift enforces SELinux by default. OSD pods may fail to access raw devices due to SELinux context mismatches.

Check for SELinux denials on a node:

```bash
ausearch -m avc -ts recent | grep rook
```

Resolution - add the correct SELinux context to the namespace:

```bash
oc annotate namespace rook-ceph \
  openshift.io/sa.scc.supplemental-groups="1000640000/10000" \
  openshift.io/sa.scc.uid-range="1000640000/10000"
```

Alternatively, set SELinux options in the CephCluster spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  security:
    keyRotation:
      enabled: true
  placement:
    all:
      tolerations:
        - effect: NoSchedule
          operator: Exists
```

## Issue 3 - OLM Subscription Failing to Upgrade

When Rook is installed via OLM, upgrades may stall if a CSV is stuck in the Pending state:

```bash
oc -n rook-ceph get csv
```

If a CSV is stuck in `Pending` or `Installing`:

```bash
oc -n rook-ceph describe csv rook-ceph-operator.vX.Y.Z | tail -30
```

Resolution - delete the stuck installplan and let OLM recreate it:

```bash
oc -n rook-ceph delete installplan -l operators.coreos.com/rook-ceph-operator.rook-ceph=""
```

## Issue 4 - CSI Driver Not Registering

On OpenShift, the CSI node registrar may fail due to the kubelet socket path difference:

```bash
kubectl -n rook-ceph logs daemonset/rook-ceph-csi-rbdplugin \
  -c driver-registrar
```

If you see errors about socket paths, update the CSI driver config:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-ceph-operator-config
  namespace: rook-ceph
data:
  ROOK_CSI_KUBELET_DIR_PATH: /var/lib/kubelet
```

## Issue 5 - Monitoring Integration with OpenShift Prometheus

To integrate Rook metrics with the OpenShift user workload monitoring stack:

```bash
oc -n openshift-user-workload-monitoring get pod
```

Create a ServiceMonitor for Rook in the monitoring namespace:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rook-ceph
  namespace: rook-ceph
  labels:
    team: rook
spec:
  selector:
    matchLabels:
      app: rook-ceph-mgr
  endpoints:
    - port: http-metrics
      path: /metrics
      interval: 30s
```

## Verifying Rook Health on OpenShift

Run a full health check:

```bash
oc -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
oc -n rook-ceph get pods
oc -n rook-ceph get cephcluster rook-ceph -o jsonpath='{.status.phase}'
```

## Summary

Troubleshooting Rook on OpenShift primarily involves addressing SCC permissions for Rook service accounts, handling SELinux enforcement for device access, managing OLM upgrade issues, and configuring the correct kubelet socket paths for CSI registration. Systematically checking SCC assignments, SELinux audit logs, and OLM CSV status covers the majority of OpenShift-specific Rook deployment issues.
