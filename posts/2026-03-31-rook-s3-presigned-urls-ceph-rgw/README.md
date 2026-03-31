# How to Use S3 Presigned URLs with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, Presigned URL, Security, Storage

Description: Generate and use S3 presigned URLs with Ceph RGW to grant temporary, secure access to private objects for downloads, uploads, and browser-based workflows.

---

## Overview

Presigned URLs grant temporary access to a specific S3 object without requiring the requester to have AWS/Ceph credentials. They are essential for allowing browser-based file downloads or direct uploads from client applications without exposing your credentials or making buckets public.

## Generate a Presigned GET URL with AWS CLI

```bash
aws s3 presign s3://my-private-bucket/reports/annual-report.pdf \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph \
  --expires-in 3600
```

This returns a URL valid for 1 hour. Anyone with the URL can download the object.

## Generate Presigned URLs with boto3

**GET URL (download)**:

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    config=Config(signature_version="s3v4", s3={"addressing_style": "path"}),
)

url = s3.generate_presigned_url(
    ClientMethod="get_object",
    Params={"Bucket": "my-private-bucket", "Key": "reports/annual-report.pdf"},
    ExpiresIn=3600,
)
print(url)
```

**PUT URL (upload)**:

```python
url = s3.generate_presigned_url(
    ClientMethod="put_object",
    Params={
        "Bucket": "my-private-bucket",
        "Key": "uploads/user-avatar.png",
        "ContentType": "image/png",
    },
    ExpiresIn=900,  # 15 minutes
)
print(url)
```

## Use Presigned POST for Browser Uploads

Presigned POST is more flexible than presigned PUT for browser forms:

```python
presigned_post = s3.generate_presigned_post(
    Bucket="my-private-bucket",
    Key="uploads/${filename}",
    Fields={
        "Content-Type": "image/png",
    },
    Conditions=[
        ["content-length-range", 0, 10 * 1024 * 1024],  # max 10 MB
        {"Content-Type": "image/png"},
    ],
    ExpiresIn=600,
)
print(presigned_post["url"])
print(presigned_post["fields"])
```

## Use a Presigned URL from the Browser

```javascript
// Download using a presigned GET URL
const response = await fetch(presignedGetUrl);
const blob = await response.blob();
const downloadUrl = URL.createObjectURL(blob);

// Upload using a presigned PUT URL
await fetch(presignedPutUrl, {
  method: "PUT",
  headers: { "Content-Type": "image/png" },
  body: file,
});
```

## Use Presigned URL with curl

Download:

```bash
curl -o /tmp/report.pdf "<PRESIGNED_GET_URL>"
```

Upload:

```bash
curl -X PUT -T /tmp/avatar.png \
  -H "Content-Type: image/png" \
  "<PRESIGNED_PUT_URL>"
```

## Notes on Ceph RGW Presigned URLs

- Use `s3={"addressing_style": "path"}` in boto3 Config to avoid virtual-hosted-style URL issues.
- Ensure the RGW endpoint is accessible from the client requesting the URL.
- Presigned URLs cannot cross regions.

## Summary

Ceph RGW presigned URLs work identically to AWS S3 presigned URLs and are essential for secure, temporary object access in web applications. Whether for browser-based uploads, temporary downloads, or sharing private objects with third parties, presigned URLs keep credentials off the client.
