# Redis Transaction Commands Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Transaction, Command, Cheat Sheet, Atomic

Description: Complete Redis transaction commands reference covering MULTI, EXEC, DISCARD, WATCH, and optimistic locking patterns.

---

Redis transactions group multiple commands into an atomic block using MULTI/EXEC. They are not the same as SQL transactions - errors in individual commands do not cause a rollback. Here is the complete reference.

## Basic Transaction Commands

```bash
# Start a transaction block
MULTI

# Queue commands (they don't execute yet)
SET counter 1
INCR counter
SET name "Alice"

# Execute all queued commands atomically
EXEC
# Returns array of results for each command
# 1) OK
# 2) (integer) 2
# 3) OK
```

## Discard a Transaction

```bash
MULTI
SET key1 "value1"
SET key2 "value2"

# Cancel all queued commands (before EXEC)
DISCARD
```

## Error Handling in Transactions

Redis has two types of errors in transactions:

**Compile errors** (command doesn't exist or wrong arity) - abort the whole transaction:

```bash
MULTI
SET key "value"
NOTACOMMAND arg      # syntax error - transaction is aborted
EXEC                 # returns error, nothing executed
```

**Runtime errors** (wrong type operation) - only that command fails, others succeed:

```bash
MULTI
SET key "string_value"
LPUSH key "list_item"  # will fail at runtime (wrong type)
SET key2 "ok"
EXEC
# 1) OK         <- SET succeeded
# 2) WRONGTYPE  <- LPUSH failed (key is a string)
# 3) OK         <- SET key2 succeeded
```

## WATCH - Optimistic Locking

WATCH monitors keys for changes. If any watched key is modified before EXEC, the transaction is aborted:

```bash
# Watch a key before starting transaction
WATCH account:balance

# Read current value
balance = GET account:balance

MULTI
# Only executes if account:balance hasn't changed
SET account:balance [balance - 100]
EXEC
# Returns nil if transaction was aborted (someone else modified balance)
# Returns array of results if transaction succeeded
```

## Retry Pattern with WATCH

```python
import redis

r = redis.Redis()

def transfer_funds(from_account, to_account, amount):
    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(f'account:{from_account}', f'account:{to_account}')
                from_bal = float(pipe.get(f'account:{from_account}') or 0)

                if from_bal < amount:
                    raise ValueError("Insufficient funds")

                pipe.multi()
                pipe.incrbyfloat(f'account:{from_account}', -amount)
                pipe.incrbyfloat(f'account:{to_account}', amount)
                pipe.execute()
                return True

            except redis.WatchError:
                # Someone modified the watched keys, retry
                continue
```

## WATCH and UNWATCH

```bash
# Watch multiple keys
WATCH key1 key2 key3

# Stop watching (if you decide not to proceed)
UNWATCH
```

## Transaction vs. Lua Scripts

| Feature | MULTI/EXEC | Lua Script |
|---------|------------|------------|
| Atomicity | Yes | Yes |
| Conditional logic | No | Yes |
| Rollback on error | No | No |
| Optimistic locking | WATCH | Not needed |
| Performance | 1 round trip (pipeline) | 1 round trip |

For complex conditional logic within an atomic block, prefer Lua scripts over MULTI/EXEC.

## Summary

Redis transaction commands (MULTI, EXEC, DISCARD) batch commands atomically but do not support rollback. WATCH enables optimistic locking by aborting the transaction if a monitored key changes. For conditional logic inside atomic operations, Lua scripts with EVAL are more flexible than MULTI/EXEC.
