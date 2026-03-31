# How to Handle Bi-Temporal Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bi-Temporal, Temporal Data, Data Modeling, Audit Trail

Description: Implement bi-temporal data modeling in ClickHouse to track both when something was true in the real world and when it was recorded in your system.

---

## What Is Bi-Temporal Data

Bi-temporal modeling tracks two independent time axes:

1. **Valid time** - when the fact was true in the real world (business time)
2. **Transaction time** - when the fact was recorded in the database (system time)

This is essential for auditing, corrections, and answering questions like "what did our system think was true on March 1st, even if the underlying data was later corrected?"

## Real-World Use Cases

- Financial corrections: a trade was recorded on day 3 but was actually executed on day 1
- Healthcare: a diagnosis date is corrected after the initial record
- Insurance: policy effective dates differ from when policies were entered
- Compliance: reconstruct the exact state of records as they existed at any past date

## Bi-Temporal Table Design

```sql
CREATE TABLE contracts_bitemporal (
    contract_id      UInt64,
    customer_id      UInt64,
    plan_type        LowCardinality(String),
    monthly_value    Float64,
    -- Valid time (business reality)
    valid_from       Date,
    valid_to         Date DEFAULT toDate('9999-12-31'),
    -- Transaction time (when we knew about it)
    recorded_from    DateTime DEFAULT now(),
    recorded_to      DateTime DEFAULT toDateTime('9999-12-31 23:59:59'),
    -- Version for deduplication
    version          UInt32 DEFAULT 1
) ENGINE = MergeTree()
ORDER BY (contract_id, valid_from, recorded_from);
```

## Inserting Initial Records

```sql
INSERT INTO contracts_bitemporal VALUES
(101, 42, 'pro', 500.0, '2026-01-01', '9999-12-31', now(), '9999-12-31 23:59:59', 1);
```

## Correcting a Historical Record

A contract was entered incorrectly - the plan should have been 'enterprise' not 'pro' since 2026-01-01. The correction is made today (2026-03-31):

```sql
-- Close the incorrect transaction-time record
INSERT INTO contracts_bitemporal
SELECT
    contract_id, customer_id, plan_type, monthly_value,
    valid_from, valid_to,
    recorded_from,
    now() AS recorded_to,  -- close the old version now
    version
FROM contracts_bitemporal
WHERE contract_id = 101 AND recorded_to = '9999-12-31 23:59:59';

-- Insert the corrected record
INSERT INTO contracts_bitemporal VALUES
(101, 42, 'enterprise', 1000.0, '2026-01-01', '9999-12-31', now(), '9999-12-31 23:59:59', 2);
```

## Querying the Current View of Current State

What do we currently believe is true today?

```sql
SELECT contract_id, customer_id, plan_type, monthly_value
FROM contracts_bitemporal
WHERE valid_from <= today() AND valid_to >= today()
  AND recorded_to = '9999-12-31 23:59:59'
ORDER BY contract_id;
```

## Point-in-Time Audit Query

What did our system believe was true on 2026-02-15?

```sql
SELECT contract_id, plan_type, monthly_value
FROM contracts_bitemporal
WHERE valid_from <= '2026-02-15' AND valid_to >= '2026-02-15'
  AND recorded_from <= '2026-02-15'
  AND recorded_to > '2026-02-15'
ORDER BY contract_id;
```

This returns the 'pro' plan because the correction had not yet been made on that date.

## Storage Considerations

Bi-temporal tables grow with every correction. Partition by `toYYYYMM(recorded_from)` and set TTL on very old transaction-time records if audit retention requirements allow:

```sql
TTL recorded_from + INTERVAL 7 YEAR DELETE
```

## Summary

Bi-temporal data modeling in ClickHouse uses separate valid-time and transaction-time columns to support historical corrections and point-in-time audits, enabling you to answer both "what is true now" and "what did we record at any past moment" with the same table.
