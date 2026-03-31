# How to Set Up RBD Persistent Cache for Read-Heavy Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Caching, Performance

Description: Set up RBD persistent write log cache in Rook to accelerate read-heavy workloads by caching frequently accessed block data on local NVMe or PMEM storage.

---

## What is the RBD Persistent Cache

The RBD persistent write log (PWL) cache stores recently written and read data on a local fast storage device (NVMe SSD or Intel Optane PMEM). Unlike the in-process librbd cache, the persistent cache survives application restarts and provides consistent acceleration across pod restarts.

For read-heavy workloads, the persistent cache acts as a local read buffer, serving repeated reads from the local SSD rather than going over the network to Ceph OSDs.

## Prerequisites

Each Kubernetes node hosting workloads that need caching must have a local NVMe SSD or PMEM device. The persistent cache is stored as a file on this device.

Configure the cache directory on each worker node:

```bash
mkdir -p /var/cache/ceph/rbd
chmod 777 /var/cache/ceph/rbd
```

## Enabling Persistent Cache on an RBD Image

Enable the persistent cache for a specific image:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd config image set replicapool/my-vol \
    rbd_persistent_cache_mode ssd

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd config image set replicapool/my-vol \
    rbd_persistent_cache_path /var/cache/ceph/rbd

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd config image set replicapool/my-vol \
    rbd_persistent_cache_size 4294967296
```

## Global Persistent Cache Configuration

Apply persistent cache settings globally for all images:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client rbd_persistent_cache_mode ssd

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client rbd_persistent_cache_path /var/cache/ceph/rbd

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client rbd_persistent_cache_size 10737418240
```

## Kubernetes Node Configuration via DaemonSet

Prepare the cache directory on all nodes using a DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rbd-cache-prep
  namespace: rook-ceph
spec:
  selector:
    matchLabels:
      app: rbd-cache-prep
  template:
    metadata:
      labels:
        app: rbd-cache-prep
    spec:
      initContainers:
        - name: prep
          image: busybox
          command:
            - sh
            - -c
            - "mkdir -p /host/var/cache/ceph/rbd && chmod 777 /host/var/cache/ceph/rbd"
          volumeMounts:
            - name: host-root
              mountPath: /host
      containers:
        - name: pause
          image: gcr.io/google_containers/pause:3.1
      volumes:
        - name: host-root
          hostPath:
            path: /
```

## Verifying Persistent Cache Operation

Check that the cache file is created after mounting a volume:

```bash
ls -la /var/cache/ceph/rbd/
```

You should see a cache file named after the RBD image UUID.

Monitor cache statistics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd perf image stats replicapool/my-vol
```

## Cache Invalidation and Cleanup

The persistent cache is invalidated automatically when the RBD image is closed cleanly. To manually clean cache files:

```bash
rm -f /var/cache/ceph/rbd/<image-uuid>
```

Always ensure the image is unmapped before removing cache files to avoid corruption.

## Summary

RBD persistent cache accelerates read-heavy Kubernetes workloads by absorbing repeated reads from local NVMe storage instead of traversing the network to Ceph OSDs. Configuration is per-image or global, with the cache directory prepared on each node. For workloads with high read-to-write ratios and temporal locality, the persistent cache can reduce average read latency by an order of magnitude.
