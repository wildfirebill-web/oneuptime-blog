# How to Use S3 Multipart Upload with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Multipart Upload, Storage, Performance

Description: Use S3 multipart upload with Ceph RGW to efficiently upload large objects by splitting them into parallel parts, improving throughput and enabling resumable uploads.

---

## Overview

S3 multipart upload allows you to upload a large object as a set of parts, each uploaded independently. Ceph RGW fully supports this API. Multipart uploads improve throughput via parallel part uploads and enable resumable transfers when a network interruption occurs mid-upload.

## When to Use Multipart Upload

- Objects larger than 5 GB (required by S3 spec)
- Any object larger than 100 MB for better performance
- Long uploads that may be interrupted and need to resume

## Multipart Upload via AWS CLI

The AWS CLI handles multipart automatically for large files:

```bash
aws s3 cp /var/backups/large-backup.tar.gz s3://my-bucket/backups/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph \
  --expected-size 10737418240
```

Force multipart with a specific part size:

```bash
aws s3 cp /var/data/bigfile.bin s3://my-bucket/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph \
  --multipart-chunksize 64MB
```

## Manual Multipart Upload via s3api

**Step 1 - Initiate**:

```bash
UPLOAD_ID=$(aws s3api create-multipart-upload \
  --bucket my-bucket \
  --key bigfile.bin \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph \
  --query UploadId --output text)
echo "Upload ID: $UPLOAD_ID"
```

**Step 2 - Split and Upload Parts**:

```bash
split -b 64m /var/data/bigfile.bin /tmp/bigfile-part-
ETAGS=()

for i in $(seq -w 01 $(ls /tmp/bigfile-part-* | wc -l)); do
  PART_FILE="/tmp/bigfile-part-${i}"
  ETAG=$(aws s3api upload-part \
    --bucket my-bucket \
    --key bigfile.bin \
    --part-number $((10#$i)) \
    --body "$PART_FILE" \
    --upload-id "$UPLOAD_ID" \
    --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
    --profile ceph \
    --query ETag --output text)
  ETAGS+=("{\"ETag\": $ETAG, \"PartNumber\": $((10#$i))}")
done
```

**Step 3 - Complete**:

```bash
PARTS_JSON=$(printf '{"Parts": [%s]}' "$(IFS=,; echo "${ETAGS[*]}")")
aws s3api complete-multipart-upload \
  --bucket my-bucket \
  --key bigfile.bin \
  --upload-id "$UPLOAD_ID" \
  --multipart-upload "$PARTS_JSON" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Multipart Upload with boto3

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client("s3", endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
                   aws_access_key_id="myaccesskey",
                   aws_secret_access_key="mysecretkey")

config = TransferConfig(
    multipart_threshold=64 * 1024 * 1024,  # 64 MB
    max_concurrency=8,
    multipart_chunksize=64 * 1024 * 1024,
)

s3.upload_file(
    "/var/data/bigfile.bin",
    "my-bucket",
    "bigfile.bin",
    Config=config,
)
```

## List and Abort Incomplete Uploads

List in-progress uploads:

```bash
aws s3api list-multipart-uploads \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

Abort a stalled upload:

```bash
aws s3api abort-multipart-upload \
  --bucket my-bucket \
  --key bigfile.bin \
  --upload-id <UPLOAD_ID> \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

Ceph RGW's multipart upload support is fully S3-compatible and handles objects of any size efficiently. Using `boto3`'s `TransferConfig` or the AWS CLI's automatic chunking abstracts the complexity, while manual `s3api` calls give you full control for custom upload workflows.
