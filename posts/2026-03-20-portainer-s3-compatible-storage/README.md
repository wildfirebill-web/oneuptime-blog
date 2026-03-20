# How to Configure S3-Compatible Storage for Portainer Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, S3, MinIO, Object Storage, Backup

Description: Configure S3-compatible object storage (MinIO, Ceph, or AWS S3) for Portainer-managed applications and use it for container backups and data persistence.

## Introduction

S3-compatible object storage is the standard for cloud-native data storage. Portainer workloads can leverage S3-compatible storage through MinIO (self-hosted), Ceph RadosGW, or AWS S3 for backups, static assets, and application data. This guide covers deploying MinIO via Portainer and connecting applications to it.

## Part 1: Deploy MinIO via Portainer

```yaml
# minio-stack.yml - Deploy as Portainer stack
version: '3.8'

services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"    # S3 API
      - "9001:9001"    # Console UI
    environment:
      MINIO_ROOT_USER: "${MINIO_ROOT_USER:-minioadmin}"
      MINIO_ROOT_PASSWORD: "${MINIO_ROOT_PASSWORD:-minioadmin123}"
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio-createbuckets:
    image: quay.io/minio/mc:latest
    depends_on:
      minio:
        condition: service_healthy
    restart: on-failure
    entrypoint: >
      /bin/sh -c "
      mc alias set myminio http://minio:9000 minioadmin minioadmin123;
      mc mb myminio/backups || true;
      mc mb myminio/app-uploads || true;
      mc mb myminio/logs || true;
      exit 0;
      "

volumes:
  minio-data:
```

## Part 2: Configure Applications to Use S3

### Application Stack with S3 Integration

```yaml
# app-stack.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    restart: unless-stopped
    environment:
      # S3 configuration
      S3_ENDPOINT: http://minio:9000
      S3_ACCESS_KEY: "${S3_ACCESS_KEY}"
      S3_SECRET_KEY: "${S3_SECRET_KEY}"
      S3_BUCKET: app-uploads
      S3_REGION: us-east-1
      S3_USE_SSL: "false"
      S3_FORCE_PATH_STYLE: "true"  # Required for MinIO
    networks:
      - app-network
      - storage-network

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: "${DB_PASSWORD}"
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:
  storage-network:
    external: true  # Shared with MinIO

volumes:
  db-data:
```

## Part 3: Using MinIO Client (mc) for Operations

```bash
# Install MinIO client
curl https://dl.min.io/client/mc/release/linux-amd64/mc -o /usr/local/bin/mc
chmod +x /usr/local/bin/mc

# Configure mc alias
mc alias set portainer-minio \
  http://minio.example.com:9000 \
  your-access-key \
  your-secret-key

# List buckets
mc ls portainer-minio

# Create a bucket
mc mb portainer-minio/portainer-backups

# Upload files
mc cp /tmp/portainer-backup.tar.gz portainer-minio/portainer-backups/

# Set lifecycle policy (auto-delete after 30 days)
mc ilm add portainer-minio/portainer-backups \
  --expiry-days 30
```

## Part 4: Portainer Backup to S3

```bash
#!/bin/bash
# backup-portainer-to-s3.sh
# Run this script on a schedule to backup Portainer to S3

S3_BUCKET="portainer-minio/portainer-backups"
BACKUP_DATE=$(date +%Y-%m-%d-%H%M%S)
BACKUP_FILE="portainer-backup-$BACKUP_DATE.tar.gz"

echo "Creating Portainer backup..."

# Stop Portainer briefly for consistent backup
docker stop portainer

# Create backup
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar czf /backup/portainer.tar.gz -C /data .

# Start Portainer again
docker start portainer

# Upload to MinIO/S3
mc cp /tmp/portainer.tar.gz "$S3_BUCKET/$BACKUP_FILE"

# Verify upload
mc stat "$S3_BUCKET/$BACKUP_FILE"

# Clean up local file
rm /tmp/portainer.tar.gz

echo "Backup uploaded: $BACKUP_FILE"
```

## Part 5: S3 Configuration for Common Applications

### WordPress Media Storage

```yaml
# Use S3-compatible storage for WordPress uploads
services:
  wordpress:
    image: wordpress:latest
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_PASSWORD: "${DB_PASSWORD}"
      # Install WP plugin for S3: "WP Offload Media"
      # Configure via wp-config.php:
      WORDPRESS_CONFIG_EXTRA: |
        define('AS3CF_SETTINGS', serialize(array(
            'provider' => 'aws',
            'access-key-id' => '${S3_ACCESS_KEY}',
            'secret-access-key' => '${S3_SECRET_KEY}',
            'bucket' => 'wordpress-media',
            'region' => 'us-east-1',
            'endpoint' => 'http://minio:9000',
        )));
```

### Database Backup to S3

```bash
# Automated PostgreSQL backup to MinIO
docker run --rm \
  --network app-network \
  -e PGPASSWORD=dbpassword \
  -e MC_HOST_minio=http://access-key:secret-key@minio:9000 \
  --entrypoint sh \
  alpine/postgres:latest -c \
  "pg_dump -h db -U postgres myapp | gzip | mc pipe minio/db-backups/$(date +%Y%m%d).sql.gz"
```

## MinIO High Availability

```yaml
# MinIO distributed mode (4 nodes minimum)
services:
  minio1:
    image: quay.io/minio/minio
    command: server http://minio{1...4}/data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: password123
    volumes:
      - /data/minio1:/data
  # Repeat for minio2, minio3, minio4...
```

## Conclusion

S3-compatible storage via MinIO integrated with Portainer provides scalable, cloud-native object storage for self-hosted environments. Applications can store files, images, and backups using the same S3 API they would use with AWS, enabling easy migration between environments. Portainer backups to S3 ensure disaster recovery is always available.
