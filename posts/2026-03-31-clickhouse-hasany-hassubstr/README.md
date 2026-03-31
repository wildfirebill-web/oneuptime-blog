# How to Use hasAny() and hasSubstr() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, hasAny, hasSubstr, Array Containment, Subsequence, SQL

Description: Learn how hasAny() detects overlap between arrays and hasSubstr() checks for contiguous subsequences, with examples covering any-of filters and sequence detection.

---

ClickHouse provides two containment functions that go beyond the simple membership tests of `has()` and `hasAll()`. `hasAny(arr, candidates)` returns `1` if the array shares at least one element with the candidates array - the "any of these" pattern. `hasSubstr(arr, sub)` checks whether `sub` appears as a contiguous subsequence (maintaining order) inside `arr` - useful for detecting specific event patterns or ordered sub-sequences in logs. Understanding when to use each is key to writing expressive, efficient array queries.

## hasAny() - At Least One Overlap

`hasAny(arr, candidates)` returns `1` if at least one element from `candidates` is present in `arr`. It is the array equivalent of `value IN (list)` but for two array columns.

```sql
-- Does the array share any element with the candidates?
SELECT hasAny(['a', 'b', 'c'], ['b', 'd', 'e']) AS any_overlap;
-- Result: 1

-- No overlap
SELECT hasAny(['a', 'b', 'c'], ['x', 'y', 'z']) AS any_overlap;
-- Result: 0
```

## Filtering Rows with hasAny()

The primary use is "give me all rows whose tag array includes any of these values."

```sql
-- Find events tagged with any severity indicator
SELECT event_id, tags
FROM events
WHERE hasAny(tags, ['critical', 'high', 'urgent']);

-- Find users who have any one of these deprecated features still enabled
SELECT user_id
FROM user_profiles
WHERE hasAny(feature_flags, ['legacy_ui', 'old_auth', 'deprecated_export']);
```

## hasAny() vs. Multiple has() Calls

`hasAny()` is more concise and performs a single pass over the array, making it preferable to chaining multiple `has()` calls with `OR`.

```sql
-- Verbose version with OR
SELECT event_id
FROM events
WHERE has(tags, 'critical') OR has(tags, 'high') OR has(tags, 'urgent');

-- Equivalent and cleaner with hasAny()
SELECT event_id
FROM events
WHERE hasAny(tags, ['critical', 'high', 'urgent']);
```

## Counting Overlap with arrayIntersect()

When you need to know not just whether there is any overlap but also what the overlapping elements are, pair `hasAny()` for filtering with `arrayIntersect()` for inspection.

```sql
-- Show which of the priority tags each event actually has
SELECT
    event_id,
    arrayIntersect(tags, ['critical', 'high', 'medium', 'low']) AS matched_priorities
FROM events
WHERE hasAny(tags, ['critical', 'high', 'medium', 'low']);
```

## hasSubstr() - Contiguous Subsequence Detection

`hasSubstr(arr, sub)` checks whether `sub` appears as a contiguous, order-preserving subsequence within `arr`. This is different from `hasAll()`, which only checks set membership without caring about order or contiguity.

```sql
-- Does [2, 3] appear consecutively in the array?
SELECT hasSubstr([1, 2, 3, 4, 5], [2, 3]) AS has_pattern;
-- Result: 1

-- [2, 4] is not contiguous even though both are present
SELECT hasSubstr([1, 2, 3, 4, 5], [2, 4]) AS has_pattern;
-- Result: 0
```

## Detecting Event Sequence Patterns

`hasSubstr()` is well suited for detecting specific ordered event sequences in user sessions or distributed traces.

```sql
-- Find sessions where a user viewed a product and then immediately added to cart
SELECT session_id
FROM sessions
WHERE hasSubstr(event_sequence, ['product_view', 'add_to_cart']);

-- Find traces containing a specific span-to-span transition
SELECT trace_id
FROM distributed_traces
WHERE hasSubstr(span_names, ['db_query', 'cache_miss', 'db_retry']);
```

## Detecting Error Recovery Patterns

In operations data, you might want to find sequences where a failure was immediately followed by a retry.

```sql
-- Find request sequences where a timeout was immediately retried
SELECT
    client_id,
    request_sequence
FROM request_logs
WHERE hasSubstr(request_sequence, ['timeout', 'retry']);
```

## Combining hasAny() and hasSubstr() for Richer Conditions

You can combine both functions in a single `WHERE` clause to express complex conditions.

```sql
-- Sessions that contain a checkout sequence AND have at least one error tag
SELECT session_id
FROM sessions
WHERE hasSubstr(event_sequence, ['add_to_cart', 'checkout_start', 'payment'])
  AND hasAny(session_tags, ['error', 'exception', 'timeout']);
```

## Using hasAny() for Multi-Tenant Permission Checks

When each resource has a list of required roles and each user has a list of granted roles, `hasAny()` checks whether the user qualifies.

```sql
-- Resources the user can access based on role overlap
SELECT
    resource_id,
    resource_name
FROM resources
WHERE hasAny(required_roles, (
    SELECT granted_roles FROM users WHERE user_id = 42
));
```

## Segment Analysis with hasAny()

In marketing or product analytics, users are often assigned to multiple segments. `hasAny()` is the natural way to pull a cohort.

```sql
-- Count users belonging to any high-value segment
SELECT count() AS high_value_users
FROM user_segments
WHERE hasAny(segments, ['whales', 'power_users', 'enterprise']);
```

## Summary

`hasAny()` and `hasSubstr()` extend ClickHouse's array containment toolkit beyond what `has()` and `hasAll()` provide. `hasAny()` efficiently implements "any of these values" membership tests, replacing verbose `OR` chains with a single clean function call. `hasSubstr()` detects ordered, contiguous sub-sequences within an array, making it the right tool for event-pattern matching in session logs, distributed traces, and any sequence-based analysis. Use `arrayIntersect()` alongside `hasAny()` when you also need to see which elements matched.
