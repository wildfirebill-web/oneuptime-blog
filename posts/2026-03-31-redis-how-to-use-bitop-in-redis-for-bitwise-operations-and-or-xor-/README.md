# How to Use BITOP in Redis for Bitwise Operations (AND, OR, XOR, NOT)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bitwise, BITOP, AND, OR, XOR, NOT

Description: Learn how to use BITOP in Redis to perform AND, OR, XOR, and NOT bitwise operations across multiple string keys for user analytics and flag operations.

---

## What Is BITOP

`BITOP` performs bitwise operations between Redis strings and stores the result in a destination key. It supports AND, OR, XOR between multiple keys, and NOT on a single key. Strings of different lengths are zero-padded to match the longest input.

## Syntax

```text
BITOP AND destkey key [key ...]
BITOP OR  destkey key [key ...]
BITOP XOR destkey key [key ...]
BITOP NOT destkey key
```

- `destkey` - where the result is stored
- `key [key ...]` - input string keys (for AND, OR, XOR); single key for NOT

Returns the size of the resulting string (in bytes).

## Basic Usage

### BITOP OR - Users Active on Day 1 OR Day 2

```bash
# Day 1: users 0, 1, 3 were active (bit positions)
redis-cli SETBIT day1 0 1
redis-cli SETBIT day1 1 1
redis-cli SETBIT day1 3 1

# Day 2: users 1, 2, 3 were active
redis-cli SETBIT day2 1 1
redis-cli SETBIT day2 2 1
redis-cli SETBIT day2 3 1

# Union: users active on either day
redis-cli BITOP OR active_either day1 day2
redis-cli BITCOUNT active_either
```

```text
(integer) 4
```

Users 0, 1, 2, 3 are active in the union.

### BITOP AND - Users Active on Both Days

```bash
redis-cli BITOP AND active_both day1 day2
redis-cli BITCOUNT active_both
```

```text
(integer) 2
```

Users 1 and 3 were active on both days.

### BITOP XOR - Users Active on Only One Day

```bash
redis-cli BITOP XOR active_one_day day1 day2
redis-cli BITCOUNT active_one_day
```

```text
(integer) 2
```

Users 0 (day1 only) and 2 (day2 only).

### BITOP NOT - Invert Bits

```bash
# NOT flips all bits in the string
redis-cli BITOP NOT not_day1 day1
```

## Practical Use Cases

### Daily Active User Analytics

```python
import redis
import datetime

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def mark_user_active(user_id, date=None):
    if date is None:
        date = datetime.date.today().isoformat()
    key = f'active:{date}'
    r.setbit(key, user_id, 1)

def get_weekly_active_users(start_date, days=7):
    keys = []
    for i in range(days):
        d = start_date + datetime.timedelta(days=i)
        keys.append(f'active:{d.isoformat()}')

    r.bitop('OR', 'active:weekly', *keys)
    return r.bitcount('active:weekly')

def get_daily_retained_users(day1_key, day2_key):
    r.bitop('AND', 'retained', day1_key, day2_key)
    return r.bitcount('retained')

# Mark active users
for uid in [1, 5, 42, 100, 200]:
    mark_user_active(uid, datetime.date(2026, 3, 30))
for uid in [5, 42, 300, 400]:
    mark_user_active(uid, datetime.date(2026, 3, 31))

# Users active on both days (retention)
retained = get_daily_retained_users('active:2026-03-30', 'active:2026-03-31')
print(f"Retained users: {retained}")  # 2 (user 5 and 42)
```

### Feature Flag Intersection

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Users with each feature enabled (bit = user ID)
r.setbit('feature:dark_mode', 1, 1)
r.setbit('feature:dark_mode', 2, 1)
r.setbit('feature:dark_mode', 5, 1)

r.setbit('feature:beta_dashboard', 2, 1)
r.setbit('feature:beta_dashboard', 3, 1)
r.setbit('feature:beta_dashboard', 5, 1)

# Users who have BOTH features
r.bitop('AND', 'feature:both', 'feature:dark_mode', 'feature:beta_dashboard')
count = r.bitcount('feature:both')
print(f"Users with both features: {count}")  # 2 (user 2 and 5)

# Users with EITHER feature
r.bitop('OR', 'feature:any', 'feature:dark_mode', 'feature:beta_dashboard')
count = r.bitcount('feature:any')
print(f"Users with any feature: {count}")  # 4 (1, 2, 3, 5)
```

### Cohort Analysis

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Cohorts as bitmaps (user ID = bit position)
r.setbit('cohort:signup_jan', 1, 1)
r.setbit('cohort:signup_jan', 2, 1)
r.setbit('cohort:signup_jan', 3, 1)

r.setbit('cohort:purchased', 2, 1)
r.setbit('cohort:purchased', 4, 1)
r.setbit('cohort:purchased', 5, 1)

r.setbit('cohort:returned', 2, 1)
r.setbit('cohort:returned', 3, 1)

# January signups who purchased AND returned
r.bitop('AND', 'cohort:jan_converted', 'cohort:signup_jan', 'cohort:purchased', 'cohort:returned')
converted = r.bitcount('cohort:jan_converted')
total_jan = r.bitcount('cohort:signup_jan')
print(f"Jan signups who converted: {converted}/{total_jan}")  # 1/3 (only user 2)
```

### XOR for Change Detection

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Snapshot before and after a batch job
r.setbit('snapshot:before', 1, 1)
r.setbit('snapshot:before', 2, 1)
r.setbit('snapshot:before', 5, 1)

r.setbit('snapshot:after', 1, 1)
r.setbit('snapshot:after', 3, 1)  # user 3 added
r.setbit('snapshot:after', 5, 1)  # user 2 removed

# Changed users (added or removed)
r.bitop('XOR', 'snapshot:changed', 'snapshot:before', 'snapshot:after')
changes = r.bitcount('snapshot:changed')
print(f"Users changed: {changes}")  # 2 (user 2 removed, user 3 added)
```

## Summary

`BITOP` performs set-like operations over bitmap keys using bitwise AND, OR, XOR, and NOT, storing the result in a destination key. It is the foundation of bitmap-based user analytics where each bit represents a user ID: OR for union (any day active), AND for intersection (all days active), XOR for symmetric difference (changed status), and NOT for complement. Combined with `BITCOUNT`, it enables real-time DAU, retention, and cohort analytics at scale.
