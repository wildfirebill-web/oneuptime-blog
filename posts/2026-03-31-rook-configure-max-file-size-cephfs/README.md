# How to Configure Maximum File Size in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Quota, Storage

Description: Learn how to configure the maximum file size limit in CephFS to prevent oversized files from impacting cluster performance and capacity.

---

## Overview

CephFS allows administrators to set a maximum file size limit on a filesystem. This is useful when you need to prevent clients from writing extremely large files that could exhaust storage capacity or degrade performance. The limit is enforced at the MDS level when clients attempt to extend file sizes beyond the configured threshold.

## Default Behavior

By default, CephFS does not enforce a maximum file size. Files can grow to any size as long as storage capacity allows. Setting a hard cap protects the cluster from runaway writes.

## Check Current Max File Size

To view the current setting for a filesystem named `cephfs`:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs get cephfs
```

Look for the `max_file_size` field in the output. A value of `0` means unlimited.

## Set Maximum File Size

Use the `ceph fs set` command to configure the maximum file size in bytes:

```bash
# Set max file size to 1 TiB (1099511627776 bytes)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs max_file_size 1099511627776
```

You can use common byte values for other sizes:

```text
1 GiB  =  1073741824
10 GiB = 10737418240
1 TiB  = 1099511627776
```

## Verify the Setting

After applying the limit, verify it is active:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs get cephfs | grep max_file_size
```

## Client-Side Behavior

When a client attempts to write a file that would exceed the configured maximum, the write operation returns an `EFBIG` (file too large) error. Applications should handle this error gracefully by catching the appropriate OS-level exception:

```python
import errno
import os

try:
    with open("/mnt/cephfs/largefile", "wb") as f:
        f.write(data)
except OSError as e:
    if e.errno == errno.EFBIG:
        print("Write rejected: file exceeds the maximum size allowed by CephFS.")
    else:
        raise
```

## Resetting to Unlimited

To remove the limit and allow files of any size again, set `max_file_size` back to `0`:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs max_file_size 0
```

## Considerations

- The limit applies to individual file sizes, not directory or filesystem quotas.
- Existing files that already exceed the new limit will not be truncated.
- Combine this setting with directory quotas for comprehensive capacity management.

## Summary

Configuring a maximum file size in CephFS using `ceph fs set max_file_size` is a simple but effective safeguard against unbounded file growth. By enforcing this limit at the MDS level, you protect your Rook-Ceph cluster from storage exhaustion caused by oversized files, while applications receive a clear `EFBIG` error that can be handled gracefully.
