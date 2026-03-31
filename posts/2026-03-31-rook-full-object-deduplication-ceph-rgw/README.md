# How to Set Up Full Object Deduplication in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Deduplication, Storage, Efficiency, Object Storage

Description: Configure full object deduplication in Ceph RGW to reduce storage consumption by identifying and sharing identical object data across buckets.

---

Ceph RGW supports full object deduplication (dedup), which detects when two or more uploaded objects have identical content and stores the data only once, sharing the underlying RADOS object. This reduces storage usage without any application changes.

## How Deduplication Works in Ceph RGW

RGW dedup computes a fingerprint (hash) of object content at upload time. If an existing object has the same fingerprint, the new object's data reference points to the existing RADOS data object rather than duplicating it. Only metadata and index entries are created anew.

## Enabling Object Deduplication

Dedup is configured at the zone placement level and requires the fingerprint feature:

```bash
# Enable fingerprint computation for new objects
ceph config set client.rgw rgw_dedup_chunk_algo sha256
ceph config set client.rgw rgw_dedup_index_type HMAC_SHA256

# Enable dedup at the zone placement
radosgw-admin zone placement modify \
  --rgw-zone default \
  --placement-id default-placement \
  --data-extra-pool default.rgw.buckets.data.dedup
```

## Running Dedup on Existing Objects

For existing objects, use the `radosgw-admin dedup` tooling:

```bash
# Start a dedup scan across all objects in the data pool
radosgw-admin dedup \
  --pool default.rgw.buckets.data \
  --num-shards 16 \
  --chunk-pool default.rgw.buckets.data.dedup
```

Check progress:

```bash
radosgw-admin dedup status
```

## Checking Dedup Savings

After running dedup, check how much space was saved:

```bash
radosgw-admin dedup estimate \
  --pool default.rgw.buckets.data
```

The output shows estimated unique data vs total data, giving a dedup ratio.

## Verifying Deduplication with RADOS

Check that two objects share the same RADOS data object:

```bash
# Get the internal RADOS object name for bucket objects
MARKER=$(radosgw-admin bucket stats --bucket bucket-a | jq -r '.marker')
rados -p default.rgw.buckets.data stat "${MARKER}_file.txt"

MARKER2=$(radosgw-admin bucket stats --bucket bucket-b | jq -r '.marker')
rados -p default.rgw.buckets.data stat "${MARKER2}_file.txt"

# If dedup worked, both should reference the same chunk pool object
```

## Important Considerations

- Dedup works best for workloads with repeated content (backups, OS images, software packages)
- Objects must be above the minimum size threshold to be considered for dedup
- Versioned objects may complicate dedup since each version has different metadata
- Encryption (SSE) prevents dedup because encrypted content is always unique

## Monitoring Dedup Efficiency

```bash
# Check pool usage with dedup
ceph df detail | grep dedup

# Check RADOS object count vs logical object count
radosgw-admin bucket stats --bucket mybucket
```

## Summary

Ceph RGW full object deduplication reduces storage consumption by detecting identical object content and sharing the underlying RADOS data. Enable it at the zone placement level and run periodic dedup scans to process existing objects. Dedup is most effective for content that is frequently re-uploaded unchanged, such as software packages, container images, or backup datasets.
