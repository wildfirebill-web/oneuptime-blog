# How to Implement a Logging System in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Logging, Schema, Partition, Performance

Description: Learn how to design a MySQL application logging system with partitioning, JSON metadata, and efficient querying for high-volume event data.

---

## Logging System Schema

Application logs need to handle high insert volumes, efficient querying by time range or severity, and automatic data expiration. The schema uses table partitioning by time for efficient range queries and purging.

```sql
CREATE TABLE app_logs (
    id BIGINT AUTO_INCREMENT,
    log_level ENUM('DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL') NOT NULL,
    service VARCHAR(100) NOT NULL,
    message TEXT NOT NULL,
    context JSON,                     -- Structured metadata
    request_id VARCHAR(36),           -- Trace correlation ID
    user_id INT DEFAULT NULL,
    ip_address VARCHAR(45),
    created_at TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    PRIMARY KEY (id, created_at),     -- Partition key must be in PK
    INDEX idx_service_level (service, log_level, created_at),
    INDEX idx_request_id (request_id),
    INDEX idx_user (user_id, created_at)
) ENGINE=InnoDB
PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
    PARTITION p_2026_01 VALUES LESS THAN (UNIX_TIMESTAMP('2026-02-01')),
    PARTITION p_2026_02 VALUES LESS THAN (UNIX_TIMESTAMP('2026-03-01')),
    PARTITION p_2026_03 VALUES LESS THAN (UNIX_TIMESTAMP('2026-04-01')),
    PARTITION p_2026_04 VALUES LESS THAN (UNIX_TIMESTAMP('2026-05-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Inserting Log Entries

```sql
-- Insert a simple info log
INSERT INTO app_logs (log_level, service, message, request_id)
VALUES ('INFO', 'api-gateway', 'User login successful', 'req-uuid-1234');

-- Insert with structured context
INSERT INTO app_logs
    (log_level, service, message, context, request_id, user_id, ip_address)
VALUES (
    'ERROR',
    'payment-service',
    'Payment processing failed',
    JSON_OBJECT(
        'error_code', 'CARD_DECLINED',
        'amount', 99.99,
        'currency', 'USD',
        'provider', 'stripe'
    ),
    'req-uuid-5678',
    42,
    '192.168.1.100'
);
```

## Application-Level Logger

```python
import json
import mysql.connector
from datetime import datetime

class MySQLLogger:
    def __init__(self, conn):
        self.conn = conn
        self.service = 'my-service'

    def log(self, level, message, context=None, request_id=None, user_id=None):
        cursor = self.conn.cursor()
        cursor.execute("""
            INSERT INTO app_logs
                (log_level, service, message, context, request_id, user_id)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (
            level,
            self.service,
            message,
            json.dumps(context) if context else None,
            request_id,
            user_id
        ))
        self.conn.commit()

    def error(self, message, **kwargs):
        self.log('ERROR', message, **kwargs)

    def info(self, message, **kwargs):
        self.log('INFO', message, **kwargs)
```

## Querying Logs

```sql
-- Recent errors in the last hour
SELECT
    id,
    log_level,
    service,
    message,
    context,
    created_at
FROM app_logs
WHERE log_level IN ('ERROR', 'CRITICAL')
  AND created_at >= NOW() - INTERVAL 1 HOUR
ORDER BY created_at DESC
LIMIT 100;

-- Query logs for a specific request trace
SELECT log_level, service, message, created_at
FROM app_logs
WHERE request_id = 'req-uuid-5678'
ORDER BY created_at;

-- Logs with specific JSON context field
SELECT id, message, context->>'$.error_code' AS error_code, created_at
FROM app_logs
WHERE service = 'payment-service'
  AND context->>'$.error_code' = 'CARD_DECLINED'
  AND created_at >= NOW() - INTERVAL 24 HOUR;
```

## Log Volume by Service and Level

```sql
SELECT
    service,
    log_level,
    COUNT(*) AS log_count
FROM app_logs
WHERE created_at >= NOW() - INTERVAL 1 DAY
GROUP BY service, log_level
ORDER BY service, FIELD(log_level, 'CRITICAL', 'ERROR', 'WARNING', 'INFO', 'DEBUG');
```

## Partition Rotation (Automated Cleanup)

```sql
-- Drop old partition and add a new future partition
ALTER TABLE app_logs
    DROP PARTITION p_2026_01;

ALTER TABLE app_logs REORGANIZE PARTITION p_future INTO (
    PARTITION p_2026_05 VALUES LESS THAN (UNIX_TIMESTAMP('2026-06-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

## Summary

A MySQL logging system combines a structured schema with JSON context columns, table partitioning by time, and targeted indexes to handle high insert rates while enabling efficient time-range and field-based queries. Monthly partitions make dropping old data instantaneous (no row-by-row delete) and keep query scans focused on relevant time windows.
