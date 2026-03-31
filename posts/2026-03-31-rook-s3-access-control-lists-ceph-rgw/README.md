# How to Set Up S3 Access Control Lists (ACLs) in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, RGW, ACL, Security, Storage

Description: Configure S3 Access Control Lists on Ceph RGW buckets and objects to control read and write permissions for individual users and predefined groups.

---

## Overview

Ceph RGW supports S3-compatible Access Control Lists (ACLs) for both buckets and objects. ACLs let you grant specific permissions to individual users or predefined groups like `AllUsers` (public) and `AuthenticatedUsers`. While bucket policies are more flexible, ACLs are simpler for basic access control.

## Predefined ACL Canned Values

Ceph RGW supports the following canned ACLs:

| Canned ACL | Description |
|---|---|
| private | Owner has full control |
| public-read | Owner full control, all others read |
| public-read-write | Owner full control, all others read and write |
| authenticated-read | Owner full control, authenticated users read |

## Set a Canned ACL on a Bucket

```bash
aws s3api put-bucket-acl \
  --bucket my-bucket \
  --acl public-read \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set a Canned ACL on an Object

```bash
aws s3api put-object-acl \
  --bucket my-bucket \
  --key public/logo.png \
  --acl public-read \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set Object ACL at Upload Time

```bash
aws s3 cp /tmp/image.png s3://my-bucket/images/image.png \
  --acl public-read \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Grant ACL to a Specific User

Get the RGW canonical user ID:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user info --uid=bob | jq .keys[0].access_key
```

Set a custom ACL granting another user read access:

```bash
aws s3api put-bucket-acl \
  --bucket my-bucket \
  --access-control-policy '{
    "Grants": [
      {
        "Grantee": {
          "Type": "CanonicalUser",
          "ID": "<bob-canonical-id>"
        },
        "Permission": "READ"
      }
    ],
    "Owner": {
      "ID": "<owner-canonical-id>"
    }
  }' \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Get Current ACL

```bash
aws s3api get-bucket-acl \
  --bucket my-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph

aws s3api get-object-acl \
  --bucket my-bucket \
  --key images/image.png \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --profile ceph
```

## Set ACL with boto3

```python
s3.put_object_acl(
    Bucket="my-bucket",
    Key="public/file.txt",
    ACL="public-read",
)
```

## Summary

Ceph RGW's S3 ACL support enables fine-grained access control at the bucket and object level. For simple use cases, canned ACLs like `public-read` are quick to apply. For multi-user environments, explicit grants using canonical user IDs provide the precision needed to share specific resources.
