# How to Test MySQL Disaster Recovery Procedures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Disaster Recovery, Backup

Description: Learn how to test MySQL disaster recovery procedures by running structured restore drills, validating data integrity, and measuring recovery time objectives.

---

A disaster recovery plan that has never been tested is unreliable. Regular DR drills expose gaps in procedures, measure actual recovery time, and build team confidence before a real incident occurs. Testing should be scheduled, documented, and treated with the same urgency as production deployments.

## What to Test

A complete DR test validates three things:
- Backups are restorable
- Data integrity after restore matches expectations
- Recovery time meets the defined RTO (Recovery Time Objective)

## Setting Up an Isolated Test Environment

Never test restores on the production server. Use a dedicated test instance:

```bash
# Launch a test MySQL instance with Docker
docker run -d \
  --name mysql-dr-test \
  -e MYSQL_ROOT_PASSWORD=testpass \
  -e MYSQL_DATABASE=myapp_db_test \
  -p 3307:3306 \
  mysql:8.0

# Wait for it to be ready
until docker exec mysql-dr-test mysqladmin ping -uroot -ptestpass --silent; do
  sleep 2
done
```

## Test 1: Full Backup Restore

Record the start time, restore, and record the end time to measure RTO:

```bash
#!/bin/bash
START_TIME=$(date +%s)

BACKUP_FILE="/backups/mysql/myapp_db-2026-03-31.sql.gz"

gunzip < "$BACKUP_FILE" | \
  mysql -h 127.0.0.1 -P 3307 -uroot -ptestpass myapp_db_test

END_TIME=$(date +%s)
ELAPSED=$((END_TIME - START_TIME))
echo "Restore completed in ${ELAPSED}s"
```

## Test 2: Data Integrity Validation

After restoring, run validation queries to confirm the data matches expectations:

```sql
-- Check row counts match production snapshot
SELECT
  table_name,
  table_rows
FROM information_schema.tables
WHERE table_schema = 'myapp_db_test'
ORDER BY table_name;

-- Spot-check specific records
SELECT COUNT(*) FROM orders WHERE created_at >= '2026-01-01';
SELECT SUM(amount) FROM payments WHERE status = 'completed';

-- Verify referential integrity
SELECT COUNT(*)
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;  -- Should return 0
```

## Test 3: Point-in-Time Recovery

Test that binary log replay works correctly:

```bash
# Restore base backup
gunzip < /backups/mysql/myapp_db-2026-03-30.sql.gz | \
  mysql -h 127.0.0.1 -P 3307 -uroot -ptestpass myapp_db_test

# Apply binary logs to a specific point in time
mysqlbinlog \
  --stop-datetime="2026-03-31 08:00:00" \
  /backups/binlogs/mysql-bin.000100 | \
  mysql -h 127.0.0.1 -P 3307 -uroot -ptestpass myapp_db_test

# Verify a record created at 07:55 exists
mysql -h 127.0.0.1 -P 3307 -uroot -ptestpass myapp_db_test \
  -e "SELECT * FROM orders WHERE order_id = 99999;"
```

## Test 4: Cross-Region Restore

Verify the offsite copy is recoverable by restoring from the DR bucket:

```bash
aws s3 cp \
  s3://myapp-mysql-dr-eu/myapp_db/2026-03-31/myapp_db-2026-03-31.sql.gz \
  /tmp/dr-test.sql.gz \
  --region eu-west-1

gunzip < /tmp/dr-test.sql.gz | \
  mysql -h 127.0.0.1 -P 3307 -uroot -ptestpass myapp_db_test
```

## Documenting Results

Record each test run in a DR test log:

```text
Date: 2026-03-31
Test Type: Full Backup Restore
Backup File: myapp_db-2026-03-30.sql.gz
Restore Duration: 342 seconds
Row Count Match: Yes
Integrity Check: Passed
Notes: Restore from S3 DR bucket took 85s longer due to download time
```

## Scheduling Regular Tests

Run DR tests on a quarterly schedule at minimum, and after any significant change to backup procedures, schema, or database size.

## Summary

Testing MySQL disaster recovery procedures requires restoring to an isolated environment, validating data integrity with SQL checks, measuring recovery time, and testing both local and cross-region backup sources. Document every test run and schedule recurring drills to ensure procedures stay current and effective.
