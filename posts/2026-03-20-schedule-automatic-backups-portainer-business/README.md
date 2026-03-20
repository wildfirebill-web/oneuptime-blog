# How to Schedule Automatic Backups in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Backup, Automation, DevOps

Description: Learn how to configure scheduled automatic backups in Portainer Business Edition using both the built-in UI scheduler and cron-based automation.

---

Portainer Business Edition includes a built-in backup scheduling feature that automatically saves encrypted backups on a regular interval. This eliminates the need for external cron scripts to protect your Portainer configuration.

## Configure Scheduled Backups in the UI

### Step 1: Navigate to Backup Settings

1. Log in to Portainer BE as an administrator
2. Go to **Settings** in the left sidebar
3. Select **Backup & Restore**

### Step 2: Enable Scheduled Backups

In the **Scheduled Backups** section:
- Toggle **Enable scheduling** to ON
- Set the **Cron schedule** (e.g., `0 2 * * *` for 2 AM daily)
- Optionally set a **Password** to encrypt the backup archive
- Click **Save Settings**

Portainer will now automatically create encrypted backup archives on the defined schedule.

## Configure via API (Automation)

For GitOps or infrastructure-as-code workflows, configure backups via the API:

```bash
# First, log in to get a JWT token
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Configure scheduled backup: daily at 2 AM UTC with encryption
curl -X PUT \
  https://localhost:9443/api/backup/s3 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "scheduleEnabled": true,
    "cronRule": "0 2 * * *",
    "password": "your-backup-encryption-password"
  }' \
  --insecure
```

## External Cron Backup (Alternative)

If you prefer an external cron job to control backup storage:

```bash
#!/bin/bash
# /usr/local/bin/portainer-auto-backup.sh
# Schedule: 0 2 * * * /usr/local/bin/portainer-auto-backup.sh

set -e

BACKUP_DIR="/opt/portainer-backups"
PORTAINER_URL="https://localhost:9443"
ADMIN_USER="admin"
ADMIN_PASS="yourpassword"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Get auth token
TOKEN=$(curl -s -X POST \
  "${PORTAINER_URL}/api/auth" \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASS}\"}" \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Download backup
curl -s -X POST \
  "${PORTAINER_URL}/api/backup" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password":"backup-encryption-key"}' \
  --output "${BACKUP_DIR}/portainer_${DATE}.tar.gz" \
  --insecure

# Remove old backups
find "$BACKUP_DIR" -name "portainer_*.tar.gz" -mtime +"$RETENTION_DAYS" -delete

echo "[$(date)] Backup saved: ${BACKUP_DIR}/portainer_${DATE}.tar.gz"
```

```bash
# Install the cron job
chmod +x /usr/local/bin/portainer-auto-backup.sh
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/portainer-auto-backup.sh >> /var/log/portainer-backup.log 2>&1") | crontab -
```

## Verify Backups Are Running

```bash
# Check cron log
tail -20 /var/log/portainer-backup.log

# List backup files
ls -lh /opt/portainer-backups/
```

---

*Monitor your backup job execution and get alerted on failures with [OneUptime](https://oneuptime.com).*
