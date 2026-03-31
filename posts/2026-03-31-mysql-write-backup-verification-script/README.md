# How to Write a MySQL Backup Verification Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Backup, Script

Description: Write a MySQL backup verification script that checks dump file integrity, restores to a test database, and validates row counts against production.

---

A backup that has never been tested is not a backup - it is a hope. A backup verification script automates the process of validating that your MySQL backups can actually be restored and that data integrity is preserved.

## What Verification Should Cover

1. Backup file exists and is not empty or corrupt
2. File can be decompressed (if gzipped)
3. SQL can be imported without errors
4. Key table row counts match the source database
5. Schema is complete (expected tables are present)

## The Verification Script

```bash
#!/bin/bash
# mysql_verify_backup.sh

BACKUP_FILE="${1:-/backups/latest.sql.gz}"
TEST_DB="backup_verification_$(date +%s)"
PROD_DB="app_db"
MYSQL="mysql -u root -p${MYSQL_ROOT_PASSWORD}"
ISSUES=()
EXIT_CODE=0

log() { echo "[$(date '+%H:%M:%S')] $1"; }

# Step 1: Check file exists and is non-empty
if [ ! -f "$BACKUP_FILE" ]; then
  echo "CRITICAL: Backup file not found: $BACKUP_FILE"
  exit 2
fi

FILE_SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
if [ "$FILE_SIZE" -lt 1024 ]; then
  echo "CRITICAL: Backup file is suspiciously small (${FILE_SIZE} bytes)"
  exit 2
fi
log "Backup file: $BACKUP_FILE (${FILE_SIZE} bytes)"

# Step 2: Test decompression
if [[ "$BACKUP_FILE" == *.gz ]]; then
  if ! gunzip -t "$BACKUP_FILE" 2>/dev/null; then
    echo "CRITICAL: Backup file is corrupt (gunzip test failed)"
    exit 2
  fi
  log "Gzip integrity: OK"
fi

# Step 3: Create test database and restore
$MYSQL -e "CREATE DATABASE ${TEST_DB};"
log "Created test database: ${TEST_DB}"

if [[ "$BACKUP_FILE" == *.gz ]]; then
  gunzip -c "$BACKUP_FILE" | $MYSQL "$TEST_DB"
else
  $MYSQL "$TEST_DB" < "$BACKUP_FILE"
fi

if [ $? -ne 0 ]; then
  echo "CRITICAL: Restore failed"
  $MYSQL -e "DROP DATABASE IF EXISTS ${TEST_DB};"
  exit 2
fi
log "Restore: completed"

# Step 4: Compare table list
PROD_TABLES=$($MYSQL -se "SHOW TABLES FROM ${PROD_DB}" | sort)
TEST_TABLES=$($MYSQL -se "SHOW TABLES FROM ${TEST_DB}" | sort)

if [ "$PROD_TABLES" != "$TEST_TABLES" ]; then
  ISSUES+=("WARNING: Table list mismatch between production and backup")
  EXIT_CODE=1
fi

# Step 5: Compare row counts for key tables
for TABLE in users orders products; do
  PROD_COUNT=$($MYSQL -se "SELECT COUNT(*) FROM ${PROD_DB}.${TABLE}" 2>/dev/null)
  TEST_COUNT=$($MYSQL -se "SELECT COUNT(*) FROM ${TEST_DB}.${TABLE}" 2>/dev/null)
  if [ -z "$PROD_COUNT" ] || [ -z "$TEST_COUNT" ]; then continue; fi

  DIFF=$(echo "scale=2; ($PROD_COUNT - $TEST_COUNT) * 100 / $PROD_COUNT" | bc 2>/dev/null)
  log "Table ${TABLE}: prod=${PROD_COUNT}, backup=${TEST_COUNT}, diff=${DIFF}%"

  if (( $(echo "${DIFF#-} > 5" | bc -l) )); then
    ISSUES+=("WARNING: ${TABLE} row count differs by ${DIFF}% (prod=${PROD_COUNT}, backup=${TEST_COUNT})")
    EXIT_CODE=1
  fi
done

# Cleanup
$MYSQL -e "DROP DATABASE IF EXISTS ${TEST_DB};"
log "Cleaned up test database"

# Output
if [ ${#ISSUES[@]} -eq 0 ]; then
  echo "OK: Backup verified successfully"
else
  for ISSUE in "${ISSUES[@]}"; do echo "$ISSUE"; done
fi
exit $EXIT_CODE
```

## Running the Script

```bash
chmod +x mysql_verify_backup.sh
./mysql_verify_backup.sh /backups/app_db_2026-03-31.sql.gz
```

## Scheduling Verification

```text
# Verify every night at 3am after backup completes at 2am
0 3 * * * /opt/scripts/mysql_verify_backup.sh /backups/latest.sql.gz >> /var/log/backup_verify.log 2>&1
```

## Summary

Backup verification should test file integrity, decompression, importability, schema completeness, and row count consistency against production. Automate it nightly so failures are caught before you need the backup in an emergency. Drop the test database after verification to avoid consuming disk space.
