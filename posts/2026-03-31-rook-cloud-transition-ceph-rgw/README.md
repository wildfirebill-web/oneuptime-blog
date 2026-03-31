# How to Set Up Cloud Transition for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cloud, Tiering, Lifecycle, Object Storage, S3

Description: Configure cloud transition lifecycle rules in Ceph RGW to automatically move objects to external S3-compatible storage after a specified number of days.

---

Ceph RGW cloud transition allows you to automatically move objects from your local Ceph cluster to external S3-compatible storage (AWS S3, Backblaze B2, MinIO, etc.) using lifecycle rules. This is ideal for cost-optimized cold storage tiering.

## Architecture Overview

Cloud transition works via a cloud-s3 sync tier:
1. A lifecycle rule triggers after N days
2. RGW copies the object to the remote S3 bucket
3. The local object becomes a stub referencing the remote location
4. Reads of transitioned objects are transparently fetched from the remote (or restored locally)

## Step 1 - Configure the Remote Cloud Zone

Create a storage tier for the cloud target:

```bash
radosgw-admin zone create \
  --rgw-zonegroup default \
  --rgw-zone cloud-tier \
  --tier-type cloud-s3

radosgw-admin zone placement modify \
  --rgw-zone cloud-tier \
  --placement-id default-placement \
  --tier-type cloud-s3 \
  --tier-config=endpoint=https://s3.amazonaws.com,\
access_key=AKIAIOSFODNN7EXAMPLE,\
secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY,\
target_path=my-cold-storage-bucket,\
retain_head_object=true
```

The `retain_head_object=true` setting keeps the object metadata locally for fast HEAD requests.

## Step 2 - Commit the Period

```bash
radosgw-admin period update --commit
```

## Step 3 - Create a Lifecycle Policy with Cloud Transition

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket mybucket \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "transition-to-cloud",
        "Status": "Enabled",
        "Filter": {"Prefix": "archive/"},
        "Transitions": [
          {
            "Days": 30,
            "StorageClass": "GLACIER"
          }
        ]
      }
    ]
  }' \
  --endpoint-url http://your-rgw-host:7480
```

Note: RGW maps the `GLACIER` storage class to the cloud-s3 tier for transition purposes.

## Step 4 - Verify Transition Status

Check lifecycle configuration:

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket mybucket \
  --endpoint-url http://your-rgw-host:7480
```

After the transition period, check object metadata:

```bash
aws s3api head-object \
  --bucket mybucket \
  --key archive/myfile.txt \
  --endpoint-url http://your-rgw-host:7480
```

Transitioned objects show `StorageClass: GLACIER` in the response.

## Listing Transitioned Objects

```bash
aws s3api list-objects-v2 \
  --bucket mybucket \
  --prefix archive/ \
  --query 'Contents[?StorageClass==`GLACIER`]' \
  --endpoint-url http://your-rgw-host:7480
```

## Accessing Transitioned Objects

By default, transitioned objects are fetched transparently from the remote store on GET. If `retain_head_object=false`, a restore operation is needed first (see Cloud Restore).

## Summary

Ceph RGW cloud transition automates cost-optimized tiering by moving infrequently accessed objects to external S3-compatible storage using lifecycle rules. Configure a cloud-s3 zone tier, commit the period, and attach lifecycle rules to buckets. Objects are transparently served from the remote tier or require a restore depending on your configuration.
