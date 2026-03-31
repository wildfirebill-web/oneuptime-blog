# How to Configure S3 Cross-Origin Resource Sharing (CORS) in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, CORS, Storage, Web

Description: Configure S3 CORS rules on Ceph RGW buckets to allow browser-based applications to access objects directly from different origins using presigned URLs.

---

## Overview

Cross-Origin Resource Sharing (CORS) allows web browsers to make requests to a bucket from a different domain. Ceph RGW fully supports S3 CORS configuration, which is useful when building web applications that upload files directly to Ceph or access public assets hosted there.

## Set CORS Configuration via AWS CLI

Create a CORS configuration file `cors.json`:

```json
{
  "CORSRules": [
    {
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
      "AllowedOrigins": ["https://app.example.com", "http://localhost:3000"],
      "ExposeHeaders": ["ETag", "x-amz-request-id"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

Apply the CORS config:

```bash
aws s3api put-bucket-cors \
  --bucket my-web-bucket \
  --cors-configuration file://cors.json \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Retrieve CORS Configuration

```bash
aws s3api get-bucket-cors \
  --bucket my-web-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set CORS with boto3

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://rook-ceph-rgw-my-store.rook-ceph:80",
    aws_access_key_id="myaccesskey",
    aws_secret_access_key="mysecretkey",
    config=Config(s3={"addressing_style": "path"}),
)

cors_config = {
    "CORSRules": [
        {
            "AllowedHeaders": ["Authorization", "Content-Type"],
            "AllowedMethods": ["GET", "PUT"],
            "AllowedOrigins": ["https://app.example.com"],
            "ExposeHeaders": ["ETag"],
            "MaxAgeSeconds": 86400,
        }
    ]
}

s3.put_bucket_cors(Bucket="my-web-bucket", CORSConfiguration=cors_config)
```

## Test CORS with curl

Send a preflight OPTIONS request:

```bash
curl -v -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: Content-Type" \
  "http://rook-ceph-rgw-my-store.rook-ceph:80/my-web-bucket/test.txt"
```

You should see response headers like:

```text
< Access-Control-Allow-Origin: https://app.example.com
< Access-Control-Allow-Methods: PUT
< Access-Control-Allow-Headers: Content-Type
< Access-Control-Max-Age: 3600
```

## JavaScript Browser Upload Example

```javascript
const response = await fetch(presignedUrl, {
  method: "PUT",
  headers: {
    "Content-Type": file.type,
  },
  body: file,
});

if (response.ok) {
  console.log("Upload successful");
}
```

## Delete CORS Configuration

```bash
aws s3api delete-bucket-cors \
  --bucket my-web-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Summary

Ceph RGW's CORS support enables browser-based direct uploads and asset serving from web applications hosted on different origins. By configuring allowed origins, methods, and headers, you can safely expose Ceph buckets to your frontend applications.
