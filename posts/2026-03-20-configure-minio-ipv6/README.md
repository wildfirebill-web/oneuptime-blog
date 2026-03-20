# How to Configure MinIO with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, MinIO, Object Storage, S3, Cloud Storage

Description: Learn how to configure MinIO object storage server to listen on IPv6 addresses for S3-compatible API access and distributed cluster deployments.

## Start MinIO on IPv6

```bash
# Start MinIO on specific IPv6 address
MINIO_ROOT_USER=minioadmin MINIO_ROOT_PASSWORD=minioadmin \
minio server --address "[2001:db8::10]:9000" \
    --console-address "[2001:db8::10]:9001" \
    /data

# Listen on all IPv6 interfaces
minio server --address "[::]:9000" \
    --console-address "[::]:9001" \
    /data

# With TLS
MINIO_ROOT_USER=minioadmin MINIO_ROOT_PASSWORD=minioadmin \
minio server \
    --address "[2001:db8::10]:9000" \
    --console-address "[2001:db8::10]:9001" \
    --certs-dir /etc/minio/certs \
    /data
```

## Systemd Service Configuration

```ini
# /etc/systemd/system/minio.service

[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# /etc/minio/minio.env

MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin

# Listen on IPv6 address
MINIO_OPTS="--address [2001:db8::10]:9000 --console-address [2001:db8::10]:9001"

MINIO_VOLUMES=/data
```

## Distributed MinIO Cluster with IPv6

```bash
# Start distributed MinIO on 4 nodes over IPv6
MINIO_ROOT_USER=minioadmin MINIO_ROOT_PASSWORD=minioadmin \
minio server \
    --address "[2001:db8::10]:9000" \
    http://[2001:db8::10]/data \
    http://[2001:db8::11]/data \
    http://[2001:db8::12]/data \
    http://[2001:db8::13]/data
```

## Verify and Test

```bash
# Check listening ports
ss -6 -tlnp | grep minio
# Expected: [2001:db8::10]:9000 (S3 API), [2001:db8::10]:9001 (console)

# Test S3 API health check
curl -6 http://[2001:db8::10]:9000/minio/health/live

# Configure mc (MinIO Client) for IPv6
mc alias set myminio http://[2001:db8::10]:9000 minioadmin minioadmin

# List buckets
mc ls myminio

# Create a bucket
mc mb myminio/mybucket

# Upload a file
mc cp /tmp/testfile.txt myminio/mybucket/
```

## AWS CLI with MinIO over IPv6

```bash
# Configure AWS CLI to use MinIO over IPv6
aws configure set default.s3.endpoint_url "http://[2001:db8::10]:9000"

# Or use --endpoint-url flag
aws --endpoint-url http://[2001:db8::10]:9000 s3 ls

# Create bucket
aws --endpoint-url http://[2001:db8::10]:9000 s3 mb s3://mybucket

# Upload file
aws --endpoint-url http://[2001:db8::10]:9000 s3 cp /tmp/file.txt s3://mybucket/

# List objects
aws --endpoint-url http://[2001:db8::10]:9000 s3 ls s3://mybucket/
```

## Python boto3 Client over IPv6

```python
import boto3
from botocore.config import Config

# Connect to MinIO via IPv6 using boto3
s3_client = boto3.client(
    's3',
    endpoint_url='http://[2001:db8::10]:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin',
    region_name='us-east-1',  # Required but not used by MinIO
    config=Config(signature_version='s3v4')
)

# Create bucket
s3_client.create_bucket(Bucket='mybucket')

# Upload a file
s3_client.upload_file('/tmp/test.txt', 'mybucket', 'test.txt')

# List objects
response = s3_client.list_objects_v2(Bucket='mybucket')
for obj in response.get('Contents', []):
    print(f"Object: {obj['Key']}, Size: {obj['Size']}")

# Download a file
s3_client.download_file('mybucket', 'test.txt', '/tmp/downloaded.txt')
```

## Summary

Configure MinIO for IPv6 with `--address "[2001:db8::10]:9000"` and `--console-address "[2001:db8::10]:9001"` startup flags. Use `[::]:9000` for all interfaces. For the systemd service, set these in `MINIO_OPTS` environment variable. Test with `curl -6 http://[2001:db8::10]:9000/minio/health/live`. Configure `mc` alias with `http://[2001:db8::10]:9000`. Python boto3 clients use `endpoint_url='http://[2001:db8::10]:9000'`.
