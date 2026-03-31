# How to Configure Ceph Manager Modules for NFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, NFS, Manager, Kubernetes

Description: Learn how to enable and configure Ceph Manager modules required for NFS in Rook, including the NFS orchestrator and dashboard integration.

---

## The Role of Ceph Manager in NFS

Ceph NFS-Ganesha in a Rook cluster is orchestrated through the Ceph Manager daemon. The Manager runs modules that handle NFS export configuration storage in RADOS and expose NFS management capabilities via the Ceph dashboard and CLI. Without the correct Manager modules enabled, the `CephNFS` operator cannot fully function and the `ceph nfs` CLI commands will not work.

## Required Manager Modules

The NFS feature in Ceph requires two Manager modules to be enabled:

1. `nfs` - Manages NFS export configuration stored in RADOS objects
2. `orchestrator` - Allows the Manager to interact with Rook to create NFS pods

Check which modules are currently enabled:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module ls
```

## Enabling NFS Manager Modules

Enable the required modules from the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable nfs

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable rook
```

The `rook` orchestrator module tells Ceph Manager to use Rook for NFS pod management. Verify that the orchestrator is set correctly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph orch status
```

Expected output:

```text
Backend: rook
Available: Yes
Paused: No
```

## Configuring NFS Pool via Manager

After enabling the NFS module, configure which Ceph pool it uses to store export configs:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph nfs cluster create my-nfs \
  --pool my-fs-data0 \
  --namespace nfs-ns
```

This creates the NFS cluster context in the Manager, which aligns with the `rados` config in the `CephNFS` CR.

## Verifying Module Status

Check that the NFS module is active and healthy:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module ls | grep -A5 '"enabled_modules"'
```

Also verify Manager is active with no standby-only issues:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr dump | grep '"active_name"'
```

## Troubleshooting Module Issues

If `ceph nfs` commands return errors like `no handler found`, the NFS module may not be loaded. Force-enable it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mgr module enable nfs --force
```

Also check Manager logs:

```bash
kubectl -n rook-ceph logs \
  $(kubectl -n rook-ceph get pod -l app=rook-ceph-mgr -o name | head -1) \
  | grep -i nfs
```

## Summary

Ceph Manager modules are the backbone of NFS orchestration in Rook. Enabling the `nfs` and `rook` modules via the Ceph CLI connects the Manager's export management layer to Rook's pod orchestration. Always verify module status and orchestrator backend after enabling them, and check Manager logs when NFS commands fail unexpectedly.
