# How to Export and Import Portainer Configuration - Config

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Backup, Export, Import, Configuration, Migration

Description: A comprehensive guide to exporting and importing Portainer configuration for backups, migrations, and environment cloning.

## Overview

Portainer stores all configuration - endpoints, users, stacks, registries, and settings - in a BoltDB database. Exporting this configuration allows you to back up your setup, migrate to a new server, or clone environments. This guide covers all methods for exporting and importing Portainer configuration.

## Prerequisites

- Portainer CE or Business Edition
- Docker access to the Portainer container
- Admin credentials

## Understanding Portainer's Data Storage

```text
portainer_data volume contains:
├── portainer.db          # BoltDB database (main config)
├── certs/                # TLS certificates
│   ├── cert.pem
│   └── key.pem
├── chisel/               # Reverse tunnel config
└── tls/                  # Custom CA certificates
```

## Method 1: Export via Docker Volume (CE and BE)

```bash
# Stop Portainer for consistent backup

docker stop portainer

# Export the entire data volume
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/portainer-config-$(date +%Y%m%d).tar.gz -C /data .

# Restart Portainer
docker start portainer

# Verify the export
ls -lh portainer-config-*.tar.gz
tar tzf portainer-config-$(date +%Y%m%d).tar.gz
```

## Method 2: Export via Portainer BE API

```bash
PORTAINER_URL="https://portainer.example.com:9443"
TOKEN="your-api-token"

# Export configuration (BE only)
curl -X POST \
  "${PORTAINER_URL}/api/backup" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"password": "ExportPassword123"}' \
  --output portainer-export-$(date +%Y%m%d).tar.gz

echo "Export complete: $(ls -lh portainer-export-*.tar.gz)"
```

## Method 3: Export Specific Configuration Components

### Export Stacks

```bash
# List all stacks via API
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${PORTAINER_URL}/api/stacks" | jq '.[].Name'

# Export a specific stack's compose file
STACK_ID=1
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${PORTAINER_URL}/api/stacks/${STACK_ID}/file" \
  | jq -r '.StackFileContent' > stack-${STACK_ID}-compose.yml
```

### Export Endpoints/Environments

```bash
# Export environment list
curl -s -H "Authorization: Bearer ${TOKEN}" \
  "${PORTAINER_URL}/api/endpoints" \
  | jq '[.[] | {Id, Name, URL, Type, PublicURL}]' \
  > portainer-endpoints.json
```

## Method 4: Import Configuration on New Instance

```bash
# Stop new Portainer if running
docker stop portainer-new

# Restore the volume from backup
docker run --rm \
  -v portainer_data_new:/data \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /data && tar xzf /backup/portainer-config-20260320.tar.gz"

# Start Portainer with the restored volume
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data_new:/data \
  portainer/portainer-ce:latest
```

## Method 5: Import BE Backup via API

```bash
# Restore from BE backup file
curl -X POST \
  "${PORTAINER_URL}/api/restore" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@portainer-export-20260320.tar.gz" \
  -F "password=ExportPassword123"
```

## Configuration Export Checklist

| Component | Included in Volume Backup | Included in BE API Backup |
|---|---|---|
| Users and teams | Yes | Yes |
| Environments/endpoints | Yes | Yes |
| Stacks | Yes | Yes |
| Registries | Yes | Yes |
| Access controls | Yes | Yes |
| TLS certificates | Yes | Yes |
| Settings | Yes | Yes |
| Container data | No | No |

## Automating Exports

```bash
#!/bin/bash
# /usr/local/bin/portainer-export.sh
BACKUP_DIR="/opt/portainer-backups"
RETENTION_DAYS=30

mkdir -p "${BACKUP_DIR}"

docker stop portainer

docker run --rm \
  -v portainer_data:/data \
  -v "${BACKUP_DIR}":/backup \
  alpine \
  tar czf /backup/portainer-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

docker start portainer

# Remove old backups
find "${BACKUP_DIR}" -name "portainer-*.tar.gz" \
  -mtime +${RETENTION_DAYS} -delete

echo "Export complete: $(ls -lh ${BACKUP_DIR}/portainer-*.tar.gz | tail -1)"
```

## Conclusion

Portainer configuration can be exported using volume-level backups (available in CE and BE) or the native API backup (BE only). Volume backups capture everything including certificates and the full BoltDB database. Always test your import process in a staging environment before relying on it for production recovery. Storing exports off-site (S3, NFS) ensures availability even if the host fails.
