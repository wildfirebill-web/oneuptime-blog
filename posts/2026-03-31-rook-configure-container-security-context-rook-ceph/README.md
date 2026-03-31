# How to Configure Container Security Context for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Security, Container, Kubernetes

Description: Configure container security contexts for Rook-Ceph pods to drop unnecessary capabilities, set non-root user execution, and harden OSD and operator containers against privilege escalation.

---

## Security Contexts in Rook-Ceph

Kubernetes security contexts control what a container can do at the OS level. For Ceph, there is a tension between security hardening and the genuine need for certain privileges (OSD pods need device access). The goal is to grant only what is strictly necessary.

## Operator Security Context

The Rook operator needs fewer privileges than OSD pods. Harden it via the Helm values:

```yaml
operator:
  securityContext:
    runAsNonRoot: true
    runAsUser: 2016
    runAsGroup: 2016
    fsGroup: 2016
    seccompProfile:
      type: RuntimeDefault
```

For the container-level context:

```yaml
containers:
  - name: rook-ceph-operator
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

## OSD Container Security Context

OSD pods require block device access, which requires specific capabilities. Configure the minimum required set:

```yaml
containers:
  - name: osd
    securityContext:
      privileged: true
      runAsUser: 0
      capabilities:
        add:
          - SYS_ADMIN
          - MKNOD
        drop:
          - NET_ADMIN
          - SYS_PTRACE
```

When using deviceFilter in Rook with PVC-based OSDs, reduce OSD privileges further:

```yaml
spec:
  storage:
    storageClassDeviceSets:
      - name: set1
        count: 3
        preparePlacement: {}
        volumeClaimTemplates:
          - metadata:
              name: data
```

## CSI Plugin Security Context

The CSI node plugin needs elevated access to mount devices. Configure its security context via Rook Helm values:

```yaml
csi:
  cephFSPluginVolume:
    - name: host-sys
      hostPath:
        path: /sys
        type: ""
  rbdPluginVolume:
    - name: host-dev
      hostPath:
        path: /dev
        type: ""
```

The CSI plugin containers use `privileged: true` to call mount syscalls. This is unavoidable for the node plugin.

## Seccomp Profiles

Apply a seccomp profile to non-privileged Ceph components:

```yaml
spec:
  template:
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

For OSD pods, use a custom seccomp profile that allows only the required syscalls:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {"names": ["open", "read", "write", "close", "stat", "fstat"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["ioctl", "mmap", "munmap", "mprotect"], "action": "SCMP_ACT_ALLOW"},
    {"names": ["io_setup", "io_submit", "io_getevents", "io_destroy"], "action": "SCMP_ACT_ALLOW"}
  ]
}
```

## Read-Only Root Filesystem

For the Rook operator and MGR pods, enable read-only root filesystem:

```yaml
containers:
  - name: rook-ceph-mgr
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
      - name: tmp-dir
        mountPath: /tmp
volumes:
  - name: tmp-dir
    emptyDir: {}
```

## Automated Security Scanning

Use kube-bench or Trivy to verify security context compliance:

```bash
kubectl run trivy-scan --image=aquasec/trivy:latest \
  --restart=Never \
  -- k8s --report summary namespace rook-ceph
```

## Summary

Container security contexts for Rook-Ceph follow the principle of minimum privilege: the operator drops all capabilities and runs as non-root, while OSD pods retain only the device-access capabilities they require. Applying seccomp profiles and read-only root filesystems to non-privileged components provides defense-in-depth without impacting storage functionality.
