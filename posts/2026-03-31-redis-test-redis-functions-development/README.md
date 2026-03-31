# How to Test Redis Functions in Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Function, Testing, Development, Lua

Description: Learn practical techniques for testing Redis Functions locally using redis-cli, unit tests with mock environments, and integration test patterns before production deployment.

---

Testing Redis Functions before deploying to production prevents costly bugs in server-side logic. This guide covers a testing workflow from quick manual checks to automated integration tests.

## Setting Up a Local Test Environment

Start a local Redis 7.0+ instance with Docker for isolated testing:

```bash
docker run -d --name redis-test -p 6379:6379 redis:7-alpine

# Verify Redis Functions support
docker exec redis-test redis-cli FUNCTION LIST
```

## Writing a Testable Function

Structure your function with clear inputs and outputs:

```lua
#!lua name=cart_lib

local function add_to_cart(keys, args)
  local cart_key = keys[1]
  local item_id = args[1]
  local quantity = tonumber(args[2])

  if not item_id or not quantity or quantity <= 0 then
    return redis.error_reply('INVALID_ARGS')
  end

  redis.call('HINCRBY', cart_key, item_id, quantity)
  local total_items = redis.call('HLEN', cart_key)
  return total_items
end

redis.register_function('add_to_cart', add_to_cart)
```

## Manual Testing with redis-cli

Load the function and run FCALL directly:

```bash
# Load the function
redis-cli FUNCTION LOAD "$(cat cart_lib.lua)"

# Test happy path
redis-cli FCALL add_to_cart 1 cart:user:42 "item:99" 3
# Expected: (integer) 1

# Test adding another item
redis-cli FCALL add_to_cart 1 cart:user:42 "item:55" 1
# Expected: (integer) 2

# Test invalid quantity
redis-cli FCALL add_to_cart 1 cart:user:42 "item:99" -1
# Expected: (error) INVALID_ARGS
```

## Automated Testing with Python

Use `redis-py` to write repeatable test cases:

```python
import redis
import pytest

@pytest.fixture
def r():
    client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    # Load function before tests
    with open('cart_lib.lua', 'r') as f:
        client.function_load(f.read(), replace=True)
    client.flushdb()
    yield client
    client.close()

def test_add_single_item(r):
    result = r.fcall('add_to_cart', 1, 'cart:test', 'item:1', 2)
    assert result == 1

def test_add_multiple_items(r):
    r.fcall('add_to_cart', 1, 'cart:test', 'item:1', 1)
    result = r.fcall('add_to_cart', 1, 'cart:test', 'item:2', 3)
    assert result == 2

def test_invalid_quantity_raises_error(r):
    with pytest.raises(redis.ResponseError, match='INVALID_ARGS'):
        r.fcall('add_to_cart', 1, 'cart:test', 'item:1', -5)
```

Run the tests:

```bash
pip install redis pytest
pytest test_cart_functions.py -v
```

## Testing Edge Cases

Always test boundary conditions and error paths:

```bash
# Empty keys
redis-cli FCALL add_to_cart 0

# Missing arguments
redis-cli FCALL add_to_cart 1 cart:test

# Large quantities
redis-cli FCALL add_to_cart 1 cart:test "item:1" 9999999
```

## Integration Testing in CI

Add function tests to your CI pipeline:

```bash
#!/bin/bash
set -e

# Start Redis
docker run -d --name redis-ci -p 6379:6379 redis:7-alpine
sleep 1

# Load and test
redis-cli FUNCTION LOAD "$(cat cart_lib.lua)"
pytest tests/test_redis_functions.py

# Cleanup
docker stop redis-ci && docker rm redis-ci
```

## Summary

Testing Redis Functions requires loading them into a local Redis 7.0+ instance and calling them with `FCALL`. Automate tests using `redis-py` and `pytest` to cover happy paths, error cases, and edge conditions. Integrate these tests into CI to catch regressions before production deployment.
