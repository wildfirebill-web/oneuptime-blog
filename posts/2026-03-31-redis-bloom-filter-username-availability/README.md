# How to Use Redis Bloom Filters for Username Availability Check

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, Username, Registration

Description: Use a Redis Bloom Filter to instantly reject taken usernames during registration without hitting the database for every availability check.

---

Username availability checks at registration time traditionally require a database query per request. Under heavy traffic, this creates unnecessary load. A Redis Bloom Filter acts as a fast pre-filter: if the filter says a username is definitely not taken, skip the database lookup entirely.

## How This Works

- All taken usernames are added to the Bloom filter at startup and as new registrations occur
- Before a database lookup, check the Bloom filter
- If the filter returns false (definitely not taken), skip the DB query and proceed
- If the filter returns true (possibly taken), fall through to the database to confirm

False positives (filter says taken when it isn't) result in an extra DB check. False negatives are impossible - the filter never says a username is available when it isn't.

## Setup

```bash
docker run -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Creating the Username Filter

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_username_filter(capacity: int = 10_000_000,
                            error_rate: float = 0.001):
    # 0.1% false positive rate means ~1 in 1000 available names
    # trigger an unnecessary DB check - acceptable tradeoff
    try:
        r.execute_command('BF.RESERVE', 'usernames', error_rate, capacity)
        print(f"Created Bloom filter: capacity={capacity}, "
              f"error_rate={error_rate}")
    except Exception:
        print("Filter already exists")

create_username_filter()
```

## Seeding from Existing Users

Load all existing usernames at startup:

```python
def seed_from_database(username_list: list,
                         batch_size: int = 10000):
    total = 0
    for i in range(0, len(username_list), batch_size):
        batch = [u.lower() for u in username_list[i:i + batch_size]]
        if batch:
            r.execute_command('BF.MADD', 'usernames', *batch)
            total += len(batch)
    print(f"Seeded {total} usernames into Bloom filter")

# Simulate loading from DB
existing_users = ["alice", "bob", "charlie", "dave"]
seed_from_database(existing_users)
```

## Username Availability Check

```python
import time

def check_username_availability(username: str,
                                  db_lookup_fn=None) -> dict:
    normalized = username.lower().strip()

    if len(normalized) < 3 or len(normalized) > 30:
        return {"available": False, "reason": "invalid_length"}

    start = time.monotonic()

    # Step 1: Bloom filter check (microseconds)
    in_filter = r.execute_command('BF.EXISTS', 'usernames', normalized)

    if not in_filter:
        # Definitely not taken - skip DB query
        return {
            "available": True,
            "source": "bloom_filter",
            "latency_ms": round((time.monotonic() - start) * 1000, 2)
        }

    # Step 2: DB confirmation (only when filter says possibly taken)
    if db_lookup_fn:
        exists_in_db = db_lookup_fn(normalized)
        return {
            "available": not exists_in_db,
            "source": "database",
            "latency_ms": round((time.monotonic() - start) * 1000, 2)
        }

    # Without DB callback, treat filter hit as unavailable
    return {"available": False, "source": "bloom_filter_conservative"}

# Simulate a DB lookup
def mock_db_lookup(username: str) -> bool:
    existing = {"alice", "bob", "charlie", "dave"}
    return username in existing

print(check_username_availability("alice", mock_db_lookup))
print(check_username_availability("zephyr_42", mock_db_lookup))
```

## Registering a New User

Add the username to the filter at registration time:

```python
def register_user(username: str) -> bool:
    normalized = username.lower().strip()
    check = check_username_availability(normalized, mock_db_lookup)

    if not check["available"]:
        return False

    # Insert into database first...
    # then update the Bloom filter
    r.execute_command('BF.ADD', 'usernames', normalized)
    return True
```

## Filter Statistics

```python
def get_filter_info() -> dict:
    result = r.execute_command('BF.INFO', 'usernames')
    info = dict(zip(result[0::2], result[1::2]))
    return {
        "capacity": info.get('Capacity'),
        "size_bytes": info.get('Size'),
        "num_filters": info.get('Number of filters'),
        "items_inserted": info.get('Number of items inserted'),
        "expansion_rate": info.get('Expansion rate')
    }

print(get_filter_info())
```

## Summary

Redis Bloom Filters eliminate database queries for username availability checks in the majority of cases. By seeding the filter at startup and adding new usernames on registration, you achieve microsecond response times for clearly available names. Only potential conflicts trigger a database round-trip, dramatically reducing DB load during peak registration periods.
