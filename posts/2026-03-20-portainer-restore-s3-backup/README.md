# How to Restore Portainer from an S3 Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Backup, S3, AWS, Restore, Recovery

Description: Restore Portainer Business Edition from an S3 backup using the built-in restore interface or manual S3 download and volume restoration process.

## Introduction

When Portainer Business Edition is configured for S3 backups, restoring is a streamlined process — download the backup from S3 and use Portainer's restore feature, or manually download and extract to the data volume. This guide covers both approaches.

## Prerequisites

- Access to the S3 bucket containing Portainer backups
- AWS CLI or S3-compatible CLI configured
- Portainer Business Edition or a clean Portainer CE installation
- The backup encryption password

## Step 1: List Available Backups in S3

```bash
# List all Portainer backups in S3
aws s3 ls s3://my-portainer-backups/portainer/ --recursive

# List with human-readable sizes and sorted by date
aws s3 ls s3://my-portainer-backups/portainer/ \
  --recursive --human-readable \
  | sort -k 1,2

# Find the most recent backup
aws s3 ls s3://my-portainer-backups/portainer/ --recursive | \
  sort -k 1,2 | tail -1
```

## Step 2: Download the Backup from S3

```bash
# Download the most recent backup
LATEST=$(aws s3 ls s3://my-portainer-backups/portainer/ --recursive | \
  sort -k 1,2 | tail -1 | awk '{print $4}')

echo "Downloading: $LATEST"

aws s3 cp "s3://my-portainer-backups/$LATEST" /tmp/portainer-restore.tar.gz

# Verify download
ls -lh /tmp/portainer-restore.tar.gz
```

## Step 3: Restore via Portainer UI

If Portainer BE is freshly installed on the target server:

1. Open Portainer at `http://your-host:9000`
2. On the initial setup screen, select **Restore Portainer from backup**
3. Upload the downloaded backup file
4. Enter the backup encryption password
5. Click **Restore**

For an existing running Portainer BE:

1. Go to **Settings** → **Backup**
2. In the **Restore** section, upload the backup file
3. Enter the password
4. Confirm the restore (this will overwrite current config)

## Step 4: Restore via Manual Volume Extraction

For more control, or if restoring to Portainer CE:

```bash
# Note: CE backup files from BE may not include all BE-specific data
# But core configuration (environments, stacks, users) will restore

# Step 1: Stop Portainer
docker stop portainer
docker rm portainer

# Step 2: Back up existing data (just in case)
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar czf /backup/portainer-before-restore.tar.gz -C /data .

# Step 3: Remove existing data volume
docker volume rm portainer_data

# Step 4: Create new volume
docker volume create portainer_data

# Step 5: Extract backup
# Note: S3 backups from Portainer BE may be in a specific format
# Try direct extraction first
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine sh -c "tar xzf /backup/portainer-restore.tar.gz -C /data 2>&1 || echo 'Direct extraction failed - may need decryption'"
```

## Step 5: Handle Encrypted Backups

Portainer BE encrypts backups with the password you set:

```bash
# If the backup is encrypted, you need to decrypt first
# Portainer uses AES-256-CBC encryption

# Method A: Use Portainer's built-in restore (handles decryption automatically)
# Method B: Decrypt manually with OpenSSL

# The encryption format used by Portainer is AES with the password
# Use the restore UI which handles this automatically

# For manual decryption (advanced):
# The Portainer backup format includes metadata about the encryption
# Best to use the Portainer restore UI
```

## Step 6: Restore Using AWS CLI Automation

```bash
#!/bin/bash
# restore-portainer-from-s3.sh

S3_BUCKET="my-portainer-backups"
S3_PREFIX="portainer/"
PORTAINER_CONTAINER="portainer"
VOLUME_NAME="portainer_data"
TEMP_DIR="/tmp/portainer-restore"

echo "=== Portainer S3 Restore ==="

# Find latest backup
echo "Finding latest backup in S3..."
LATEST_KEY=$(aws s3 ls "s3://$S3_BUCKET/$S3_PREFIX" --recursive | \
  sort -k 1,2 | tail -1 | awk '{print $4}')

if [ -z "$LATEST_KEY" ]; then
  echo "ERROR: No backups found in s3://$S3_BUCKET/$S3_PREFIX"
  exit 1
fi

BACKUP_DATE=$(echo "$LATEST_KEY" | grep -o '[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}')
echo "Latest backup: $LATEST_KEY (date: $BACKUP_DATE)"

# Confirm restoration
read -p "Restore from this backup? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
  echo "Aborted."
  exit 0
fi

# Create temp directory
mkdir -p "$TEMP_DIR"

# Download backup
echo "Downloading backup..."
aws s3 cp "s3://$S3_BUCKET/$LATEST_KEY" "$TEMP_DIR/backup.tar.gz"

echo "Backup downloaded: $(du -sh $TEMP_DIR/backup.tar.gz | cut -f1)"

# Stop Portainer
echo "Stopping Portainer..."
docker stop "$PORTAINER_CONTAINER"
docker rm "$PORTAINER_CONTAINER"

# Backup current data before overwriting
echo "Creating safety backup of current data..."
docker run --rm \
  -v "$VOLUME_NAME:/data" \
  -v "/tmp:/backup" \
  alpine tar czf /backup/portainer-pre-restore.tar.gz -C /data . 2>/dev/null || true

# Remove old volume
docker volume rm "$VOLUME_NAME" 2>/dev/null || true
docker volume create "$VOLUME_NAME"

# Restore
echo "Restoring backup..."
docker run --rm \
  -v "$VOLUME_NAME:/data" \
  -v "$TEMP_DIR:/backup" \
  alpine tar xzf /backup/backup.tar.gz -C /data

# Verify
echo "Restored files:"
docker run --rm -v "$VOLUME_NAME:/data" alpine ls -la /data/

# Start Portainer
echo "Starting Portainer..."
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name "$PORTAINER_CONTAINER" \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$VOLUME_NAME:/data" \
  portainer/portainer-ce:latest

# Cleanup
rm -rf "$TEMP_DIR"

echo "Restore complete. Portainer is starting..."
echo "Allow 10-15 seconds then access: http://$(hostname -I | awk '{print $1}'):9000"
```

## Step 7: Verify Restored Data

```bash
# Wait for Portainer to start
sleep 15

# Test API availability
curl -s http://localhost:9000/api/status | jq '.Version'

# Test login
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Check what was restored
echo "Environments:"
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/endpoints | jq '.[].Name'

echo "Stacks:"
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/stacks | jq '.[].Name'
```

## Conclusion

Restoring Portainer from an S3 backup is a two-step process: download the backup from S3 and restore it either through the Portainer BE restore UI (which handles encrypted backups automatically) or by manually extracting to the data volume. Always verify the restoration by testing authentication and confirming all environments and stacks are present. Keep your backup encryption password stored securely and separately from the backups themselves.
