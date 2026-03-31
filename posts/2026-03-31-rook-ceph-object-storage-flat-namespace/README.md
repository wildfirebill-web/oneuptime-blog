# How to Understand Object Storage in Ceph (Flat Namespace, Identifiers, Metadata)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, ObjectStorage, RADOS, Storage, Kubernetes

Description: Learn how Ceph implements object storage with a flat namespace, unique identifiers, and flexible metadata through xattrs and omap key-value stores.

---

## Object Storage Model in Ceph

Ceph's object storage model differs fundamentally from hierarchical filesystems. RADOS (the underlying storage layer) presents a flat namespace of named objects within each pool. There are no directories or inodes - just objects identified by string names up to 4096 bytes long.

This flat model is what makes RADOS scalable to billions of objects. Higher-level services like RGW translate the S3/Swift hierarchy illusion (buckets, prefixes, slashes) into this flat namespace.

## Object Identifiers

Every RADOS object is uniquely identified by:
- **Pool name or ID**: The logical partition the object belongs to
- **Object name**: An arbitrary byte string (commonly a UUID, path, or structured key)

```bash
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash

# List objects in a pool
rados -p replicapool ls

# Show object details
rados -p replicapool stat my-object-key
```

Example output:

```text
replicapool/my-object-key mtime 2026-03-31 10:00:00.000000, size 4096
```

## Object Data Payload

Each object has a primary byte-stream payload. You can read and write it directly:

```bash
# Write a test object
echo "hello world" | rados -p replicapool put test-obj -

# Read the object back
rados -p replicapool get test-obj -

# Remove the object
rados -p replicapool rm test-obj
```

## Extended Attributes (Xattrs)

Objects can carry small key-value pairs called extended attributes. Ceph services use xattrs for fast metadata lookups without reading the full object payload.

```bash
# Set an xattr
rados -p replicapool setxattr test-obj content-type "application/json"

# Get an xattr
rados -p replicapool getxattr test-obj content-type

# List all xattrs
rados -p replicapool listxattrs test-obj
```

Xattrs are stored inline in BlueStore and optimized for small values (< 64 KB).

## OMap Key-Value Store

For larger structured metadata, RADOS provides OMap - a per-object key-value store backed by RocksDB. CephFS uses OMap to store directory entries. RGW uses OMap to store bucket indexes.

```bash
# List omap keys
rados -p .rgw.buckets.index listomapkeys bucket-index-object

# Get an omap value
rados -p .rgw.buckets.index getomapval bucket-index-object some-key out.bin
```

## How RGW Uses the Flat Namespace

When you create an S3 bucket `photos` and upload `vacation/beach.jpg`, RGW stores:
- The object data in a RADOS object named something like `photos.vacation/beach.jpg` or a hashed name
- The bucket index in a separate RADOS object using OMap entries
- Bucket metadata in the `.rgw.root` pool

```bash
# List RGW pools
rados lspools | grep rgw
```

## Namespace Prefixes

RADOS supports optional namespace prefixes within a pool to logically separate objects without creating separate pools. This is used by RBD to store image headers separately from data objects:

```bash
# List objects in the rbd namespace
rados -p replicapool -N rbd ls | head -10
```

## Summary

Ceph's object storage is built on a flat namespace of identified objects within pools. Each object carries a byte-stream payload, optional xattrs for small metadata, and an OMap for structured key-value data. Higher-level services like RGW, RBD, and CephFS all translate their native data models into this object abstraction, gaining the scalability and reliability of RADOS without sacrificing the semantic richness their consumers require.
