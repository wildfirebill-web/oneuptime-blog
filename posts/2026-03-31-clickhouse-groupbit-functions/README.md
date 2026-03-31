# How to Use groupBitAnd, groupBitOr, groupBitXor in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Bitwise, groupBitAnd

Description: Learn how to aggregate integer columns with bitwise AND, OR, and XOR across groups in ClickHouse using groupBitAnd, groupBitOr, and groupBitXor.

---

ClickHouse provides three aggregate functions for bitwise aggregation: `groupBitAnd`, `groupBitOr`, and `groupBitXor`. Each applies the corresponding bitwise operation across all values in a group and returns a single integer result. These functions are especially powerful when you store feature flags, permission bitmasks, or set memberships as integers and need to roll them up across rows without unpacking individual bits in application code.

## Syntax

```sql
groupBitAnd(column)
groupBitOr(column)
groupBitXor(column)
```

All three accept any integer type (UInt8, UInt16, UInt32, UInt64, Int8, Int16, Int32, Int64) and return the same type as the input. There are no extra parameters.

## groupBitOr - Union of Flags

`groupBitOr` is the most common of the three. It computes the bitwise OR across all rows in a group, effectively finding the union of all set bits.

```sql
-- Suppose each row stores a set of enabled features as a bitmask
-- Bit 0 = feature_a, Bit 1 = feature_b, Bit 2 = feature_c
SELECT
    account_id,
    groupBitOr(feature_flags) AS all_features_ever_enabled
FROM user_feature_events
GROUP BY account_id;
```

If user A has `feature_flags = 0b0011` (features a and b) and user B in the same account has `0b0110` (features b and c), the result is `0b0111` - all three features appeared at least once.

## groupBitAnd - Intersection of Flags

`groupBitAnd` returns bits that are set in every row of the group - the intersection.

```sql
-- Find permissions that ALL users in a role share
SELECT
    role_id,
    groupBitAnd(permission_mask) AS shared_permissions
FROM user_roles
GROUP BY role_id;
```

A bit appears in the result only if it is set in every user's permission mask. If any user in the role is missing bit 2, bit 2 will be 0 in the output.

```sql
-- Check which feature flags are enabled across ALL sessions for a user
SELECT
    user_id,
    groupBitAnd(active_flags) AS flags_on_in_all_sessions
FROM user_sessions
WHERE session_date >= today() - 7
GROUP BY user_id;
```

## groupBitXor - Parity / Toggle Detection

`groupBitXor` returns bits where the count of set rows is odd. It is useful for detecting whether a flag was toggled an even or odd number of times.

```sql
-- Detect users where a toggle flag changed an odd number of times
-- (net result: flag is currently on)
SELECT
    user_id,
    groupBitXor(toggle_flag) AS net_toggle_state
FROM flag_change_events
GROUP BY user_id;
```

```sql
-- Detect data inconsistencies: XOR of a checksum column should be 0
-- if every insert was paired with a matching delete
SELECT
    batch_id,
    groupBitXor(row_checksum) AS parity_check
FROM audit_log
GROUP BY batch_id
HAVING parity_check != 0;
```

## Working with Individual Bits

Combine these functions with `bitAnd` and bitwise arithmetic to inspect or extract specific bits from the aggregated result.

```sql
-- Check if permission bit 2 (value 4) is set for any user in each role
SELECT
    role_id,
    groupBitOr(permission_mask) AS union_perms,
    bitAnd(groupBitOr(permission_mask), 4) > 0 AS any_user_has_bit2
FROM user_roles
GROUP BY role_id;
```

```sql
-- Check if ALL users in a role have read permission (bit 0 = 1)
SELECT
    role_id,
    bitAnd(groupBitAnd(permission_mask), 1) AS all_have_read
FROM user_roles
GROUP BY role_id;
```

## Feature Flag Use Case - Full Example

```sql
-- Table: user_plan_features (user_id UInt64, plan_id UInt32, flags UInt32)
-- Bit 0: export, Bit 1: api_access, Bit 2: sso, Bit 3: audit_log

-- For each plan, find: which features does any user have,
-- which do all users have, and net toggle state
SELECT
    plan_id,
    bin(groupBitOr(flags))  AS any_user_flags,
    bin(groupBitAnd(flags)) AS all_user_flags,
    bin(groupBitXor(flags)) AS xor_flags
FROM user_plan_features
GROUP BY plan_id;
```

## Permissions Rollup Example

```sql
-- Aggregate effective permissions for a team across all member roles
SELECT
    team_id,
    groupBitOr(effective_permission_mask) AS team_effective_permissions
FROM team_members
JOIN user_roles USING (user_id)
GROUP BY team_id;
```

## Summary

`groupBitOr`, `groupBitAnd`, and `groupBitXor` bring bitwise aggregation directly into ClickHouse SQL, eliminating the need to fetch rows and compute bitmask logic in application code. Use `groupBitOr` for union of flags, `groupBitAnd` for intersection, and `groupBitXor` for parity or toggle-count detection. They work with any integer type and compose naturally with ClickHouse's other bitwise scalar functions like `bitAnd`, `bitNot`, and `bin`.
