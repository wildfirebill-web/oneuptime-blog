# How to Configure MinIO Object Storage with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MinIO, Object Storage, S3, Kubernetes, Distributed Storage

Description: Configure MinIO object storage server to listen on IPv6 addresses, including standalone and distributed modes, S3-compatible client access over IPv6, and Kubernetes deployment with IPv6 services.

## Introduction

MinIO is a high-performance S3-compatible object storage server that supports IPv6 natively. Configuring MinIO for IPv6 involves setting the listen address to include IPv6 interfaces and configuring the console and API endpoints. S3 clients access MinIO over IPv6 using bracket notation for the server address.

## Standalone MinIO with IPv6

```bash
# Start MinIO listening on all interfaces (IPv4 and IPv6)
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server /data \
    --address "[::]:9000" \
    --console-address "[::]:9001"

# Listen only on a specific IPv6 address
MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server /data \
    --address "[2001:db8::10]:9000" \
    --console-address "[2001:db8::10]:9001"
```

## MinIO systemd Service with IPv6

```ini
# /etc/systemd/system/minio.service

[Unit]
Description=MinIO Object Storage
After=network-online.target
Wants=network-online.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES \
    --address $MINIO_OPTS_BIND \
    --console-address $MINIO_CONSOLE_BIND

Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
# /etc/default/minio
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=securepassword
MINIO_VOLUMES="/data"
MINIO_OPTS_BIND="[::]:9000"
MINIO_CONSOLE_BIND="[::]:9001"
# Optional: public URL for presigned URLs
MINIO_SERVER_URL="http://[2001:db8::10]:9000"
MINIO_BROWSER_REDIRECT_URL="http://[2001:db8::10]:9001"
```

## Distributed MinIO with IPv6

```bash
# Start MinIO in distributed mode across IPv6 nodes
# All nodes run the same command with all server addresses listed

MINIO_ROOT_USER=minioadmin \
MINIO_ROOT_PASSWORD=minioadmin \
minio server \
    --address "[::]:9000" \
    "http://[2001:db8::10]:9000/data" \
    "http://[2001:db8::11]:9000/data" \
    "http://[2001:db8::12]:9000/data" \
    "http://[2001:db8::13]:9000/data"

# Or use hostname-based addressing (recommended for production)
minio server \
    --address "[::]:9000" \
    "http://minio{1...4}.example.com:9000/data"
# Ensure DNS resolves hostnames to IPv6 AAAA records
```

## Access MinIO over IPv6 with mc (MinIO Client)

```bash
# Install mc
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && mv mc /usr/local/bin/

# Configure mc with IPv6 endpoint
mc alias set myminio http://[2001:db8::10]:9000 minioadmin minioadmin

# List buckets
mc ls myminio

# Create a bucket
mc mb myminio/mybucket

# Upload and download files
mc cp /local/file.txt myminio/mybucket/
mc cp myminio/mybucket/file.txt /local/downloaded.txt
```

## S3 Client Libraries with IPv6 MinIO

```python
#!/usr/bin/env python3
# minio_ipv6_client.py

import boto3
from botocore.config import Config

# Connect to MinIO over IPv6 using boto3 (S3-compatible)
s3_client = boto3.client(
    's3',
    endpoint_url='http://[2001:db8::10]:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin',
    config=Config(
        signature_version='s3v4',
        # Force path-style addressing (required for MinIO)
        s3={'addressing_style': 'path'},
    ),
    region_name='us-east-1',  # MinIO ignores this but boto3 requires it
)

# List buckets
response = s3_client.list_buckets()
for bucket in response['Buckets']:
    print(f"Bucket: {bucket['Name']}")

# Upload a file
s3_client.upload_file('/local/data.csv', 'mybucket', 'data.csv')
```

## Kubernetes MinIO Deployment with IPv6 Service

```yaml
# minio-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args:
            - server
            - /data
            - --address
            - "[::]:9000"
            - --console-address
            - "[::]:9001"
          env:
            - name: MINIO_ROOT_USER
              value: minioadmin
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: password
          ports:
            - containerPort: 9000
            - containerPort: 9001

---
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  # Dual-stack service
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv6
    - IPv4
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
```

## Verify IPv6 Operation

```bash
# Check MinIO is listening on IPv6
ss -tlnp | grep 9000
# Should show [::]:9000

# Test API access over IPv6
curl -6 http://[2001:db8::10]:9000/minio/health/live
# Expected: HTTP 200

# Test S3 API
curl -6 http://[2001:db8::10]:9000 \
    -H "Authorization: AWS4-HMAC-SHA256 ..."
```

## Conclusion

MinIO listens on IPv6 by setting `--address "[::]:9000"` in the server startup command, which binds to all IPv6 (and optionally IPv4) interfaces. The MinIO client (`mc`) and S3-compatible libraries access MinIO over IPv6 using bracket notation for the endpoint URL. In distributed mode, specify IPv6 addresses or hostnames with AAAA DNS records for all peer servers. Kubernetes deployments use `ipFamilyPolicy: PreferDualStack` on services to expose MinIO on IPv6 cluster addresses. Set `MINIO_SERVER_URL` to the public IPv6 address to ensure presigned URLs are generated with the correct endpoint.
