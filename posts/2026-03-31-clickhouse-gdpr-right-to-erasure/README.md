# How to Implement GDPR Right to Erasure in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GDPR, Right to Erasure, Compliance, Data Deletion

Description: Learn how to implement GDPR right to erasure (right to be forgotten) in ClickHouse while handling the challenges of an append-only database.

---

## The Challenge: GDPR in an Append-Only Database

GDPR Article 17 grants individuals the right to have their personal data erased. ClickHouse's append-only, columnar storage makes deletion expensive - but with the right design patterns, you can implement compliant erasure without sacrificing performance.

## Option 1: Lightweight DELETE (ClickHouse 22.8+)

Lightweight deletes use a deletion mask rather than immediately rewriting parts:

```sql
-- Erase all data for a specific user
DELETE FROM user_events WHERE user_id = 'gdpr-request-user-42';
```

Verify the deletion mask is applied:

```sql
SELECT count()
FROM user_events
WHERE user_id = 'gdpr-request-user-42';
```

Force physical removal by triggering a merge:

```sql
OPTIMIZE TABLE user_events FINAL;
```

Note: lightweight deletes are eventually consistent - data is logically hidden immediately but physically removed after the next merge.

## Option 2: ALTER TABLE DELETE Mutation

For guaranteed physical removal, use a heavy mutation:

```sql
ALTER TABLE user_events
    DELETE WHERE user_id = 'gdpr-request-user-42';
```

Monitor until complete:

```sql
SELECT command, parts_to_do, is_done, latest_fail_reason
FROM system.mutations
WHERE table = 'user_events'
  AND command LIKE '%gdpr-request-user-42%'
ORDER BY create_time DESC;
```

## Option 3: Data Pseudonymization

For irreversible anonymization instead of deletion, overwrite identifying fields with a hash:

```sql
ALTER TABLE user_events
    UPDATE
        user_id      = 'ERASED',
        email        = 'ERASED',
        ip_address   = toIPv4('0.0.0.0'),
        user_agent   = 'ERASED'
    WHERE user_id = 'gdpr-request-user-42';
```

This approach preserves aggregate analytics while removing personal data.

## Option 4: Partition Drop for Bulk Erasure

If personal data is partitioned by user or time and you need to erase an entire partition:

```sql
ALTER TABLE user_events DROP PARTITION '202301';
```

This is instant and irreversible - use with caution.

## Building an Erasure Request Workflow

```sql
-- Track erasure requests
CREATE TABLE gdpr_erasure_requests (
    request_id      String,
    user_id         String,
    requested_at    DateTime,
    status          LowCardinality(String),  -- pending, processing, completed
    completed_at    Nullable(DateTime),
    tables_affected Array(String)
) ENGINE = MergeTree()
ORDER BY (requested_at, user_id);
```

Update status after erasure:

```sql
ALTER TABLE gdpr_erasure_requests
    UPDATE
        status       = 'completed',
        completed_at = now()
    WHERE request_id = 'req-12345';
```

## Verifying Erasure Completion

After erasure, run a verification query and store the result:

```sql
SELECT count() AS remaining_rows
FROM user_events
WHERE user_id = 'gdpr-request-user-42';
-- Should return 0
```

Document the verification timestamp in the erasure request record for compliance evidence.

## Summary

GDPR right to erasure in ClickHouse is achievable through lightweight deletes for fast logical erasure, heavy mutations for guaranteed physical removal, or pseudonymization for analytics-preserving anonymization. Build an erasure request tracking table to document compliance, and always verify erasure completion before closing requests.
