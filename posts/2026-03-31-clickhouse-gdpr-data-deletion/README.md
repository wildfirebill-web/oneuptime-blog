# How to Implement GDPR Data Deletion in ClickHouse

Author: [oneuptime](https://www.github.com/oneuptime)

Tags: ClickHouse, GDPR, Privacy, Database, Compliance

Description: Learn how to implement GDPR-compliant right-to-erasure workflows in ClickHouse using lightweight deletes, mutations, and TTL-based automatic removal.

## Introduction

The General Data Protection Regulation (GDPR) grants EU residents the right to erasure - commonly called the "right to be forgotten." When a user exercises this right, you must delete or irreversibly anonymize all personal data you hold about them. This is technically challenging in ClickHouse because its MergeTree engine is optimized for append-only workloads and does not support transactional row-level deletes by design.

This guide covers every deletion mechanism ClickHouse offers, when to use each one, and how to build a practical erasure pipeline that satisfies GDPR obligations.

## Understanding ClickHouse Deletion Semantics

ClickHouse offers three mechanisms for removing data:

1. **Lightweight DELETE** - marks rows logically deleted; physical removal happens during subsequent merges.
2. **ALTER TABLE ... DELETE (mutation)** - rewrites entire parts to remove matching rows; heavyweight but thorough.
3. **TTL expressions** - automatically removes rows or columns when a time condition is met.

None of these are instant. You must account for the asynchronous nature of physical deletion in your compliance documentation.

## Lightweight DELETE (Recommended Starting Point)

Introduced in ClickHouse 22.8, lightweight deletes are the least disruptive option for targeted row removal.

```sql
-- Enable lightweight deletes (required in some versions)
SET allow_experimental_lightweight_delete = 1;

-- Delete all rows for a specific user
DELETE FROM user_events WHERE user_id = 'usr-8821a3bc';

-- Verify rows are logically gone immediately
SELECT count() FROM user_events WHERE user_id = 'usr-8821a3bc';
-- Returns: 0 (logical delete is immediate)
```

Physically, the rows remain on disk until the next background merge. If physical removal is required before a specific deadline, force a merge after the delete.

```sql
-- Force immediate merge to physically remove deleted rows
OPTIMIZE TABLE user_events FINAL;
```

## ALTER TABLE Mutation Delete

For older ClickHouse versions or when you need guaranteed physical deletion across all replicas, use a mutation.

```sql
-- Submit a delete mutation
ALTER TABLE user_events DELETE WHERE user_id = 'usr-8821a3bc';

-- Check mutation status
SELECT
    mutation_id,
    command,
    create_time,
    is_done,
    parts_to_do
FROM system.mutations
WHERE table = 'user_events'
ORDER BY create_time DESC
LIMIT 5;
```

Monitor until `is_done = 1` before reporting the erasure as complete.

```sql
-- Wait for completion in a loop (run from application code)
SELECT count() FROM system.mutations
WHERE table = 'user_events'
  AND is_done = 0
  AND command LIKE '%usr-8821a3bc%';
-- When this returns 0, the mutation is complete
```

## Handling Erasure Requests Across Multiple Tables

Personal data is usually scattered across several tables. Build a systematic erasure procedure.

```sql
-- Create a deletion request log
CREATE TABLE gdpr_erasure_requests
(
    request_id   UUID DEFAULT generateUUIDv4(),
    user_id      String,
    requested_at DateTime DEFAULT now(),
    completed_at Nullable(DateTime),
    status       Enum8('pending' = 1, 'in_progress' = 2, 'completed' = 3, 'failed' = 4)
)
ENGINE = MergeTree()
ORDER BY (requested_at, user_id);

-- Insert an erasure request
INSERT INTO gdpr_erasure_requests (user_id, status)
VALUES ('usr-8821a3bc', 'pending');
```

Then run the erasure across all relevant tables.

```sql
-- Step 1: Delete from events table
DELETE FROM user_events WHERE user_id = 'usr-8821a3bc';

-- Step 2: Delete from profile table
DELETE FROM user_profiles WHERE user_id = 'usr-8821a3bc';

-- Step 3: Delete from audit logs (if legally required)
DELETE FROM audit_logs WHERE actor_user_id = 'usr-8821a3bc';

-- Step 4: Anonymize remaining references that must be kept for legal reasons
-- (e.g., financial transaction records that must be retained 7 years)
ALTER TABLE transactions
    UPDATE
        email       = 'deleted@gdpr.invalid',
        full_name   = 'Deleted User',
        ip_address  = '0.0.0.0'
    WHERE user_id = 'usr-8821a3bc';

-- Step 5: Mark the erasure request complete
ALTER TABLE gdpr_erasure_requests
    UPDATE
        completed_at = now(),
        status       = 'completed'
    WHERE user_id = 'usr-8821a3bc'
      AND status = 'in_progress';
```

## Automatic Deletion with TTL

For data with known retention windows, use TTL expressions so ClickHouse removes data automatically without manual intervention.

```sql
-- Table with automatic 90-day TTL on personal data columns
CREATE TABLE session_logs
(
    session_id   String,
    user_id      String,
    email        String TTL event_time + INTERVAL 90 DAY,
    ip_address   String TTL event_time + INTERVAL 90 DAY,
    user_agent   String TTL event_time + INTERVAL 90 DAY,
    event_type   String,
    event_time   DateTime
)
ENGINE = MergeTree()
ORDER BY (event_time, session_id)
TTL event_time + INTERVAL 2 YEAR DELETE;
```

The column-level TTL replaces the email, ip_address, and user_agent with empty strings after 90 days, while the row-level TTL removes the entire row after 2 years.

## Partitioned Deletion for Bulk Erasure

When large volumes of old data must be removed at a known boundary (for example, all data older than 3 years), use partition drops instead of mutations. Dropping a partition is instant and does not require a full rewrite.

```sql
-- Create a table partitioned by month
CREATE TABLE web_analytics
(
    user_id     String,
    page_url    String,
    referrer    String,
    event_time  DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id);

-- Drop an entire month of data instantly
ALTER TABLE web_analytics DROP PARTITION 202301;

-- List available partitions
SELECT DISTINCT partition
FROM system.parts
WHERE table = 'web_analytics'
  AND active = 1
ORDER BY partition;
```

For GDPR, partition-based deletion only works when the partition boundary aligns with the retention policy. It is most effective for "delete everything older than X months" policies rather than individual user deletion.

## Verifying Deletion

Document the verification steps in your erasure workflow so you can demonstrate compliance.

```sql
-- Confirm no rows remain for the deleted user
SELECT
    'user_events'    AS table_name,
    count()          AS remaining_rows
FROM user_events
WHERE user_id = 'usr-8821a3bc'
UNION ALL
SELECT
    'user_profiles',
    count()
FROM user_profiles
WHERE user_id = 'usr-8821a3bc'
UNION ALL
SELECT
    'session_logs',
    count()
FROM session_logs
WHERE user_id = 'usr-8821a3bc';
```

All counts should return 0. Log these results and store them alongside the erasure request record.

## Limitations and Compliance Notes

- Lightweight deletes and mutations are asynchronous. If you need physical removal within a hard deadline (some DPA interpretations require deletion within 30 days), schedule `OPTIMIZE TABLE FINAL` after each deletion batch.
- Replicated tables propagate mutations to all replicas automatically, but propagation takes time. Check `system.mutations` on each replica.
- ClickHouse does not support multi-statement transactions. If your erasure spans many tables, a failure mid-way leaves partial state. Use the erasure request log to resume from the last completed step.
- Data in S3-backed cold storage (using `StoragePolicy`) is also affected by mutations, but the rewrite may be slower. Account for this in your SLA.

## Summary

GDPR-compliant deletion in ClickHouse requires a combination of lightweight deletes for row removal, TTL expressions for automatic retention enforcement, partition drops for bulk time-based removal, and a request tracking table to maintain an audit trail. Because physical deletion is asynchronous, pair every delete with an `OPTIMIZE TABLE FINAL` when hard deadlines apply, and always verify with a follow-up count query before closing the erasure request.
