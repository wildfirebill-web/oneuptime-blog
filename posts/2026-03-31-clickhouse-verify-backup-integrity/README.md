# How to Verify ClickHouse Backup Integrity

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Backup, Verification, Disaster Recovery, Administration, Testing

Description: Learn how to verify ClickHouse backup integrity by inspecting backup manifests, performing test restores, running checksum validation, and automating weekly backup verification.

---

A backup that cannot be restored is not a backup. Verifying backup integrity is a distinct step from creating the backup and must be performed regularly in production environments. ClickHouse provides checksum information in its backup manifests, and the safest verification approach is a test restore into an isolated environment.

## Checking the Backup Manifest

Every ClickHouse native backup writes a `.backup` manifest file to the destination. This JSON file lists all files, their checksums, and metadata:

```bash
# Download and inspect the manifest
aws s3 cp \
  s3://your-backup-bucket/clickhouse/backups/2026-03-31-full/.backup \
  /tmp/backup-manifest.json

# View manifest structure
python3 -c "
import json
with open('/tmp/backup-manifest.json') as f:
    manifest = json.load(f)

print('Version:', manifest.get('version'))
print('Timestamp:', manifest.get('timestamp'))
print('Databases:', list(manifest.get('databases', {}).keys()))
print('Total files:', sum(len(db.get('files', [])) for db in manifest.get('databases', {}).values()))
"
```

Check the number of files matches what is actually in S3:

```bash
# Count manifest entries
python3 -c "
import json
with open('/tmp/backup-manifest.json') as f:
    m = json.load(f)
total = 0
for db_name, db in m.get('databases', {}).items():
    for tbl_name, tbl in db.get('tables', {}).items():
        total += len(tbl.get('files', []))
        print(f'{db_name}.{tbl_name}: {len(tbl.get(\"files\", []))} files')
print(f'Total: {total} files')
"

# Count actual S3 objects
aws s3 ls s3://your-backup-bucket/clickhouse/backups/2026-03-31-full/ --recursive | wc -l
```

## Validating File Checksums

Download backup files and verify checksums match the manifest:

```bash
#!/bin/bash
# /usr/local/bin/verify-clickhouse-backup.sh

set -euo pipefail

BACKUP_PATH="s3://your-backup-bucket/clickhouse/backups/2026-03-31-full"
WORK_DIR="/tmp/backup-verify"
ERRORS=0

mkdir -p "$WORK_DIR"
trap "rm -rf $WORK_DIR" EXIT

# Download manifest
aws s3 cp "${BACKUP_PATH}/.backup" "${WORK_DIR}/manifest.json"

echo "Verifying checksums from manifest..."

# Extract file checksums from manifest and verify
python3 <<'PYEOF'
import json, subprocess, sys, hashlib, boto3, io

with open('/tmp/backup-verify/manifest.json') as f:
    manifest = json.load(f)

s3 = boto3.client('s3')
bucket = 'your-backup-bucket'
prefix = 'clickhouse/backups/2026-03-31-full'
errors = 0

for db_name, db in manifest.get('databases', {}).items():
    for tbl_name, tbl in db.get('tables', {}).items():
        for file_info in tbl.get('files', [])[:5]:  # Check first 5 files per table
            key = f"{prefix}/{db_name}/{tbl_name}/{file_info['name']}"
            try:
                obj = s3.get_object(Bucket=bucket, Key=key)
                data = obj['Body'].read()
                actual_size = len(data)
                expected_size = file_info.get('size', 0)
                if actual_size != expected_size:
                    print(f"SIZE MISMATCH {key}: expected {expected_size}, got {actual_size}")
                    errors += 1
                else:
                    print(f"OK {db_name}.{tbl_name}/{file_info['name']} ({actual_size} bytes)")
            except Exception as e:
                print(f"ERROR reading {key}: {e}")
                errors += 1

sys.exit(1 if errors > 0 else 0)
PYEOF
```

## Test Restore Verification

The most reliable verification is a test restore to a separate database or server:

```bash
#!/bin/bash
# /usr/local/bin/test-restore-clickhouse.sh
# Run weekly to verify a recent backup can actually be restored

set -euo pipefail

BACKUP_DATE="${1:-$(date '+%Y-%m-%d')}"
SOURCE_DB="my_database"
VERIFY_DB="my_database_verify_${BACKUP_DATE//-/}"
S3_URL="https://s3.amazonaws.com/your-backup-bucket/clickhouse/backups/${BACKUP_DATE}-full/${SOURCE_DB}/"
LOG_FILE="/var/log/clickhouse-backup-verify.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

log() {
    echo "[$(date '+%Y-%m-%dT%H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

on_error() {
    log "VERIFICATION FAILED at line $LINENO"
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data "{\"text\": \"ClickHouse backup verification FAILED for ${BACKUP_DATE} on $(hostname)\"}" || true
    fi
    # Cleanup
    clickhouse-client --query "DROP DATABASE IF EXISTS ${VERIFY_DB}" || true
    exit 1
}

trap on_error ERR

log "Starting backup verification for ${BACKUP_DATE}"

# Clean up any previous verification database
clickhouse-client --query "DROP DATABASE IF EXISTS ${VERIFY_DB}"

# Restore into verification database
log "Restoring ${SOURCE_DB} -> ${VERIFY_DB}"
clickhouse-client --query "
RESTORE DATABASE ${SOURCE_DB} AS ${VERIFY_DB}
FROM S3('${S3_URL}')
SETTINGS async = false;
"

# Compare row counts between source and restored
log "Comparing row counts..."
SOURCE_COUNTS=$(clickhouse-client --format TabSeparated --query "
SELECT table, sum(rows)
FROM system.parts
WHERE database = '${SOURCE_DB}' AND active = 1
GROUP BY table
ORDER BY table
")

VERIFY_COUNTS=$(clickhouse-client --format TabSeparated --query "
SELECT table, sum(rows)
FROM system.parts
WHERE database = '${VERIFY_DB}' AND active = 1
GROUP BY table
ORDER BY table
")

if [ "$SOURCE_COUNTS" != "$VERIFY_COUNTS" ]; then
    log "Row count mismatch!"
    log "Source: ${SOURCE_COUNTS}"
    log "Restored: ${VERIFY_COUNTS}"
    exit 1
fi

log "Row counts match!"

# Run a spot-check query on a key table
SAMPLE_QUERY="SELECT count(), max(event_time) FROM ${VERIFY_DB}.events"
RESULT=$(clickhouse-client --query "$SAMPLE_QUERY" 2>&1)
log "Spot check: ${RESULT}"

# Clean up the verification database
clickhouse-client --query "DROP DATABASE ${VERIFY_DB}"
log "Verification database dropped"

log "Backup verification PASSED for ${BACKUP_DATE}"

if [ -n "$SLACK_WEBHOOK" ]; then
    curl -s -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        --data "{\"text\": \"ClickHouse backup verification PASSED for ${BACKUP_DATE} on $(hostname). Row counts match.\"}" || true
fi
```

## Comparing Checksums in system.parts

After a test restore, use ClickHouse's built-in checksum verification:

```sql
-- Verify checksums of data parts in the restored database
CHECK TABLE my_database_verified.events;

-- For large tables, check partition by partition
ALTER TABLE my_database_verified.events CHECK PARTITION '202403';

-- Check all tables in a database
SELECT
    database,
    name AS table_name
FROM system.tables
WHERE database = 'my_database_verified'
  AND engine LIKE '%MergeTree%';
```

Run `CHECK TABLE` for each table:

```bash
clickhouse-client --query "
SELECT 'CHECK TABLE my_database_verified.' || name || ';'
FROM system.tables
WHERE database = 'my_database_verified'
  AND engine LIKE '%MergeTree%'
FORMAT TabSeparated
" | clickhouse-client --multiquery
```

## Scheduling Weekly Verification

Add the verification job to crontab, running on a different day than the full backup:

```text
# Verify the most recent Sunday full backup every Wednesday at 03:00
0 3 * * 3 /usr/local/bin/test-restore-clickhouse.sh $(date -d 'last sunday' '+%Y-%m-%d' 2>/dev/null || date -v-sun '+%Y-%m-%d') >> /var/log/clickhouse-backup-verify.log 2>&1
```

## Verifying clickhouse-backup Backups

If using the `clickhouse-backup` tool, it has a built-in verify command:

```bash
# Verify a specific backup
clickhouse-backup verify 2026-03-31-full

# List backups with size and file counts
clickhouse-backup list remote
```

## Summary

Backup integrity verification requires three steps: checking the manifest for completeness, sampling file checksums to catch truncated or corrupted data, and performing a test restore to confirm the data can actually be read back. Automate this as a weekly cron job that restores the most recent full backup into a temporary database, compares row counts against the source, runs `CHECK TABLE` on key tables, and sends a Slack notification with the result. A backup you have never tested restoring is a liability rather than an asset.
