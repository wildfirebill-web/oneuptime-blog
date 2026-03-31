# How to Use has() and hasAll() for Array Containment in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, has, hasAll, Array Containment, SQL

Description: Learn how has() checks if an array contains a specific element and hasAll() verifies all required elements are present, with practical filtering examples.

---

Checking whether an array contains a particular value is one of the most frequent operations in ClickHouse queries. The `has()` function answers the simple question "is this element in the array?" while `hasAll()` answers "does the array contain every one of these required elements?" Both return `1` (true) or `0` (false) and are directly usable in `WHERE` clauses, making them the standard tools for tag-based filtering, feature flag checks, and membership tests on array columns.

## has() - Single Element Membership Test

`has(arr, elem)` returns `1` if `elem` is present anywhere in `arr`, and `0` otherwise. The comparison is exact and type-sensitive.

```sql
-- Is 3 in the array?
SELECT has([1, 2, 3, 4, 5], 3) AS found;
-- Result: 1

-- Is 'error' in the log level array?
SELECT has(['info', 'warn', 'error'], 'debug') AS has_debug;
-- Result: 0
```

## Filtering Rows by Array Membership

The most common use is filtering a table to find rows whose array column contains a specific value.

```sql
-- Find all events tagged with 'payment'
SELECT event_id, tags
FROM events
WHERE has(tags, 'payment');

-- Find all users who have the 'beta_tester' feature flag
SELECT user_id
FROM user_profiles
WHERE has(feature_flags, 'beta_tester');
```

## hasAll() - Subset Membership Test

`hasAll(arr, subset)` returns `1` only when every element of `subset` is present in `arr`. The order does not matter; it is purely a set containment check.

```sql
-- Does the array contain both 'read' and 'write'?
SELECT hasAll(['read', 'write', 'admin'], ['read', 'write']) AS has_rw;
-- Result: 1

-- Does the array contain 'admin' and 'superuser'? (missing 'superuser')
SELECT hasAll(['read', 'write', 'admin'], ['admin', 'superuser']) AS has_both;
-- Result: 0
```

## Filtering Rows That Have All Required Tags

```sql
-- Find incidents tagged with both 'database' and 'critical'
SELECT incident_id, tags
FROM incidents
WHERE hasAll(tags, ['database', 'critical']);

-- Find users who have completed all required onboarding steps
SELECT user_id
FROM onboarding
WHERE hasAll(completed_steps, ['verify_email', 'set_password', 'add_payment']);
```

## Tag-Based Filtering for Analytics

`has()` works well for single-dimension tag filters in analytics queries.

```sql
-- Count events per day that include the 'checkout' tag
SELECT
    toDate(event_time) AS day,
    count() AS checkout_events
FROM events
WHERE has(tags, 'checkout')
GROUP BY day
ORDER BY day;
```

## Feature Flag Checks at Query Time

Rather than joining to a separate feature-flag table, you can store flags directly in an array column and use `has()` for enrollment checks.

```sql
-- Revenue attributable to users in the 'new_pricing' experiment
SELECT
    sum(revenue) AS experiment_revenue
FROM purchases
WHERE has(user_feature_flags, 'new_pricing');

-- Compare experiment vs control group
SELECT
    has(user_feature_flags, 'new_pricing') AS in_experiment,
    count() AS users,
    sum(revenue) AS total_revenue
FROM purchases
GROUP BY in_experiment;
```

## Combining has() and hasAll() with NOT

Negating either function with `NOT` (or using `= 0`) finds rows that are missing an element or missing required elements.

```sql
-- Events that are NOT tagged as 'internal'
SELECT event_id
FROM events
WHERE NOT has(tags, 'internal');

-- Users who have NOT completed all required steps
SELECT user_id
FROM onboarding
WHERE NOT hasAll(completed_steps, ['verify_email', 'set_password', 'add_payment']);
```

## Using has() as a Computed Column

`has()` can appear in the `SELECT` list to produce a boolean column, which is useful for downstream filtering or reporting.

```sql
-- Add a flag column indicating premium plan features
SELECT
    user_id,
    feature_flags,
    has(feature_flags, 'unlimited_api') AS is_unlimited,
    has(feature_flags, 'priority_support') AS has_priority_support
FROM user_profiles;
```

## Checking Array Membership Against a Dynamic Value

The second argument to `has()` can be a column rather than a literal, enabling row-level lookups.

```sql
-- Check if each row's required_permission is in the user's granted_permissions
SELECT
    user_id,
    required_permission,
    has(granted_permissions, required_permission) AS has_permission
FROM permission_checks;
```

## Performance Consideration

`has()` performs a linear scan through the array. For very large arrays or very high query rates, consider normalizing the array into a separate table with an index. For typical tag arrays (tens to low hundreds of elements), `has()` is fast in practice.

```sql
-- Confirm expected selectivity before using has() at scale
SELECT
    count() AS total_rows,
    countIf(has(tags, 'critical')) AS critical_rows,
    countIf(has(tags, 'critical')) / count() AS selectivity
FROM events;
```

## Summary

`has()` and `hasAll()` provide clean, readable array membership tests in ClickHouse. `has()` checks for a single element and is the go-to function for tag-based `WHERE` clauses and feature flag lookups. `hasAll()` verifies that an array is a superset of a given set of required elements, making it ideal for access control checks and multi-step completion validation. Both are direct drop-in replacements for verbose `arrayExists()` expressions when you do not need a lambda.
