# How to Handle MIME Types in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, MIME, Object Storage, HTTP

Description: Configure MIME type handling in Ceph RGW to ensure proper Content-Type headers are returned for stored objects based on file extension or explicit metadata.

---

Ceph RGW can serve objects with appropriate `Content-Type` headers by mapping file extensions to MIME types. This is important for web hosting, CDN integration, and any application that reads content type from S3 responses.

## How RGW Handles MIME Types

RGW determines the `Content-Type` for a GET response in this priority order:

1. **Explicit metadata** - `Content-Type` set by the client during upload
2. **MIME type database** - RGW's internal mapping from file extension
3. **Default** - Falls back to `binary/octet-stream`

## Checking the MIME Types File Location

```bash
ceph config get client.rgw rgw_mime_types_file
```

The default path is typically `/etc/ceph/rgw_mime_types` or `/etc/mime.types`.

## Setting a Custom MIME Types File

```bash
ceph config set client.rgw rgw_mime_types_file /etc/ceph/rgw_mime_types
```

Then create the file with standard MIME mappings:

```bash
text/html             html htm
text/plain            txt text
image/jpeg            jpg jpeg
image/png             png
image/gif             gif
application/json      json
application/xml       xml
application/pdf       pdf
video/mp4             mp4
video/webm            webm
application/zip       zip
application/gzip      gz
```

## Injecting the File in Rook

Mount the MIME types file via a ConfigMap and volume mount in Rook:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rgw-mime-types
  namespace: rook-ceph
data:
  mime.types: |
    text/html html htm
    text/plain txt
    image/jpeg jpg jpeg
    image/png png
    application/json json
    application/xml xml
    video/mp4 mp4
```

Then reference it in the CephObjectStore if using custom pod templates, or mount it via a config override.

## Setting Content-Type During Upload

Always prefer to set `Content-Type` explicitly at upload time:

```bash
# AWS CLI
aws s3 cp index.html s3://my-bucket/ \
  --content-type "text/html" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc

# Python boto3
import boto3
s3 = boto3.client('s3', endpoint_url='http://rook-ceph-rgw-my-store.rook-ceph.svc')
s3.put_object(
    Bucket='my-bucket',
    Key='page.html',
    Body=open('page.html', 'rb'),
    ContentType='text/html'
)
```

## Verifying Content-Type Responses

```bash
# Check Content-Type for a stored object
curl -I http://rook-ceph-rgw-my-store.rook-ceph.svc/my-bucket/index.html \
  -H "Authorization: AWS4 ..."

# Expected
# Content-Type: text/html
```

## Summary

Ceph RGW determines `Content-Type` from explicit upload metadata, then MIME type file mappings, then defaults to `binary/octet-stream`. Configure `rgw_mime_types_file` to point to a valid MIME database, and always set `Content-Type` explicitly when uploading to ensure correct headers for web or CDN use cases.
