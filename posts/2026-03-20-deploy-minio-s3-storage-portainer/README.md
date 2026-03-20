# How to Deploy Minio (S3-Compatible Storage) via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, MinIO, S3, Object Storage, Self-Hosting, Docker

Description: Learn how to deploy MinIO, the S3-compatible object storage server, via Portainer with a production-ready Compose stack and console access.

---

MinIO provides an Amazon S3-compatible API that lets you store unstructured data at scale on your own hardware. Any application that speaks S3 can use MinIO without code changes. Portainer makes it easy to deploy and manage the instance visually.

## Prerequisites

- Portainer running
- At least 1GB RAM free
- Sufficient disk space for your objects (mount a dedicated volume or host path)

## Compose Stack

MinIO exposes two ports: `9000` for the S3 API and `9001` for the web console. The following stack maps both and stores data in a named volume:

```yaml
version: "3.8"

services:
  minio:
    image: minio/minio:latest
    restart: always
    ports:
      - "9000:9000"    # S3 API endpoint
      - "9001:9001"    # MinIO web console
    environment:
      MINIO_ROOT_USER: minioadmin          # Change this
      MINIO_ROOT_PASSWORD: minioadmin123   # Change this (min 8 chars)
    volumes:
      - minio_data:/data
    # Start MinIO server and enable the web console
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  minio_data:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `minio`.
3. Paste the YAML, update credentials, and click **Deploy the stack**.

Access the console at `http://<host>:9001` and log in with your `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`.

## Creating Buckets via mc CLI

The MinIO client `mc` can automate bucket creation. Run it as a one-off container:

```bash
# Configure an alias for your MinIO instance
docker run --rm --network host minio/mc \
  alias set local http://localhost:9000 minioadmin minioadmin123

# Create a bucket named "backups"
docker run --rm --network host minio/mc mb local/backups
```

## S3-Compatible Client Configuration

Point any AWS SDK to MinIO by overriding the endpoint:

```python
import boto3

# Connect to MinIO using the standard boto3 S3 client
s3 = boto3.client(
    "s3",
    endpoint_url="http://localhost:9000",
    aws_access_key_id="minioadmin",
    aws_secret_access_key="minioadmin123",
    region_name="us-east-1",          # Required but ignored by MinIO
)

# Upload a file to the backups bucket
s3.upload_file("backup.tar.gz", "backups", "backup-2026-03-20.tar.gz")
```

## Monitoring

Add an HTTP monitor in OneUptime pointing to `http://<host>:9000/minio/health/live`. MinIO returns `200 OK` when healthy. Alert on any non-200 response to catch storage issues immediately.
