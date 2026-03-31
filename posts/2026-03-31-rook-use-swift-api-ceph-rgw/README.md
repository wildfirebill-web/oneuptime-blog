# How to Use the Swift API with Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Swift, OpenStack, Object Storage

Description: Learn how to use the OpenStack Swift API with Ceph RGW for object storage operations including authentication, container management, and object uploads.

---

## Overview

Ceph RGW implements the OpenStack Swift API alongside the S3 API on the same endpoint. This allows Swift-native clients, including OpenStack services and Python swiftclient, to store and retrieve objects from Ceph. Swift uses a different authentication model and terminology compared to S3 (containers instead of buckets, accounts instead of users).

## Swift Terminology vs. S3

| Swift | S3 Equivalent |
|-------|--------------|
| Account | User/Tenant |
| Container | Bucket |
| Object | Object |
| Subuser | Access Key |

## Creating a Swift-Compatible Subuser

Swift authentication requires a subuser in the format `uid:subuser`:

```bash
# Create a user and subuser for Swift
radosgw-admin user create \
  --uid=swiftuser \
  --display-name="Swift User"

# Create the subuser
radosgw-admin subuser create \
  --uid=swiftuser \
  --subuser=swiftuser:main \
  --access=full

# Generate a Swift secret key
radosgw-admin key create \
  --uid=swiftuser \
  --subuser=swiftuser:main \
  --key-type=swift \
  --gen-secret

# Retrieve the Swift secret
radosgw-admin user info --uid=swiftuser | jq '.swift_keys'
```

## Authenticating with Swift v1

Swift v1 auth uses a simple HTTP header exchange:

```bash
# Get the auth token and storage URL
curl -i http://rgw-host:80/auth/1.0 \
  -H "X-Auth-User: swiftuser:main" \
  -H "X-Auth-Key: YOUR_SWIFT_SECRET"

# Response headers include:
# X-Auth-Token: TOKEN_VALUE
# X-Storage-Url: http://rgw-host:80/swift/v1
```

## Using swiftclient CLI

```bash
# Install swiftclient
pip install python-swiftclient

# Set environment variables
export ST_AUTH=http://rgw-host:80/auth/1.0
export ST_USER=swiftuser:main
export ST_KEY=YOUR_SWIFT_SECRET

# List containers (buckets)
swift list

# Create a container
swift post mycontainer

# Upload an object
swift upload mycontainer localfile.txt

# Download an object
swift download mycontainer localfile.txt --output download.txt

# List objects in a container
swift list mycontainer
```

## Using Python swiftclient Library

```python
import swiftclient

conn = swiftclient.Connection(
    authurl='http://rgw-host:80/auth/1.0',
    user='swiftuser:main',
    key='YOUR_SWIFT_SECRET',
    auth_version='1'
)

# Create a container
conn.put_container('mycontainer')

# Upload an object
with open('file.txt', 'rb') as f:
    conn.put_object('mycontainer', 'file.txt', f)

# Download an object
headers, obj = conn.get_object('mycontainer', 'file.txt')
with open('downloaded.txt', 'wb') as f:
    f.write(obj)

# List objects
headers, objects = conn.get_container('mycontainer')
for obj in objects:
    print(obj['name'], obj['bytes'])
```

## Object Metadata and Bulk Operations

```bash
# Add metadata to an object
swift post mycontainer file.txt \
  --header "X-Object-Meta-Author: Alice"

# Bulk delete objects
swift delete mycontainer file1.txt file2.txt

# Copy an object
swift copy --destination /destcontainer/newfile.txt mycontainer file.txt
```

## Summary

Ceph RGW's Swift API compatibility allows OpenStack-native clients to use Ceph for object storage without modification. Create Swift subusers via `radosgw-admin`, authenticate using Swift v1 auth, and use standard swiftclient commands or the Python library for container and object operations. Swift and S3 users can coexist on the same Ceph cluster with independent credentials.
