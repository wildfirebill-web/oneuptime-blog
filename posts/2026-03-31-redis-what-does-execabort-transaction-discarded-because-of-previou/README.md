# What Does 'EXECABORT Transaction discarded because of previous errors' Mean

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error, EXECABORT, Transaction, MULTI, EXEC, Troubleshooting

Description: Learn why Redis returns EXECABORT when executing a transaction, how it differs from runtime errors, and how to write reliable Redis transactions with proper error handling.

---

## What Is the EXECABORT Error

When you run `EXEC` to execute a Redis transaction and there were command errors during the `MULTI` block (errors at queuing time, not at execution time), Redis discards the entire transaction and returns:

```text
(error) EXECABORT Transaction discarded because of previous errors.
```

This means the transaction was never executed - no commands ran.

## EXECABORT vs Runtime Errors in Transactions

Redis transactions have two types of errors:

**Type 1 - Command errors before EXEC (causes EXECABORT)**

These are errors detected when the command is queued, such as:
- Wrong number of arguments
- Unknown command name
- Syntax errors

```bash
MULTI
SET key1 value1
UNKNOWNCOMMAND arg1    # Error at queue time
SET key2 value2
EXEC
# (error) EXECABORT Transaction discarded because of previous errors.
```

**Type 2 - Runtime errors during EXEC (transaction continues)**

These are errors that can only be detected at execution time, such as:
- WRONGTYPE errors (wrong data type)
- Out-of-range errors

```bash
MULTI
SET key1 "hello"
LPUSH key1 item1      # Type error, but queued successfully
SET key2 value2
EXEC
# 1) OK
# 2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
# 3) OK
# key2 WAS set - transactions don't roll back on runtime errors
```

## When Does EXECABORT Occur

### Syntax Error in Queued Command

```bash
MULTI
SET foo           # Wrong number of arguments
EXEC
# (error) ERR wrong number of arguments for 'set' command
# EXEC returns:
# (error) EXECABORT Transaction discarded because of previous errors.
```

### Unknown Command

```bash
MULTI
NOTACOMMAND arg
EXEC
# (error) EXECABORT Transaction discarded because of previous errors.
```

### Sending Commands on an Errored Transaction State

Once a command errors during queuing, subsequent commands can still be queued, but EXEC will always return EXECABORT.

## How to Handle EXECABORT in Application Code

### Python (redis-py)

```python
import redis

r = redis.Redis()

def safe_transaction():
    pipe = r.pipeline()
    try:
        pipe.multi()
        pipe.set('key1', 'value1')
        pipe.set('key2', 'value2')
        results = pipe.execute()
        return results
    except redis.exceptions.ExecAbortError as e:
        print(f"Transaction was discarded: {e}")
        # Handle error - possibly retry or raise
        raise
    except redis.exceptions.ResponseError as e:
        print(f"Command error in transaction: {e}")
        raise
```

### Node.js (ioredis)

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function safeTransaction(key1, value1, key2, value2) {
  const pipeline = redis.multi();
  pipeline.set(key1, value1);
  pipeline.set(key2, value2);

  try {
    const results = await pipeline.exec();
    // results is an array of [error, result] pairs
    results.forEach(([err, result], index) => {
      if (err) {
        console.error(`Command ${index} failed: ${err.message}`);
      }
    });
    return results;
  } catch (err) {
    console.error('Transaction failed:', err.message);
    throw err;
  }
}
```

## Correct Transaction Pattern

### WATCH + MULTI + EXEC (Optimistic Locking)

```python
import redis

r = redis.Redis()

def transfer_balance(src_key, dst_key, amount):
    with r.pipeline() as pipe:
        while True:
            try:
                # Watch keys for changes
                pipe.watch(src_key, dst_key)

                src_balance = int(pipe.get(src_key) or 0)
                dst_balance = int(pipe.get(dst_key) or 0)

                if src_balance < amount:
                    pipe.reset()
                    raise ValueError("Insufficient balance")

                # Begin transaction
                pipe.multi()
                pipe.decrby(src_key, amount)
                pipe.incrby(dst_key, amount)
                pipe.execute()
                break  # Transaction succeeded

            except redis.WatchError:
                # Another client modified the keys - retry
                continue
```

### Validate Commands Before Queuing

```python
def validated_transaction(operations):
    """
    operations: list of (command, *args) tuples
    Validates commands before creating transaction
    """
    r = redis.Redis()

    # Validate first
    for command, *args in operations:
        if not hasattr(r, command.lower()):
            raise ValueError(f"Unknown Redis command: {command}")
        # Additional argument count checks could go here

    # Execute transaction
    pipe = r.pipeline()
    pipe.multi()
    for command, *args in operations:
        getattr(pipe, command.lower())(*args)

    try:
        return pipe.execute()
    except redis.exceptions.ExecAbortError:
        raise RuntimeError("Transaction aborted due to queued command error")
```

## Checking Transaction State

You can check if a transaction is in an error state by inspecting the pipeline state before calling EXEC:

```bash
MULTI
SET key1       # Error - wrong arg count
# Error at queue time

# The pipeline is now poisoned - DISCARD and start over
DISCARD
```

Always use `DISCARD` to clean up an errored transaction before starting a new one on the same connection.

## Summary

EXECABORT means a Redis transaction was discarded because one or more commands had errors at queue time (before EXEC was called), such as wrong argument counts or unknown commands. Unlike runtime errors during EXEC (which allow other commands to succeed), EXECABORT cancels the entire transaction. Fix it by validating commands before queuing them, catching ExecAbortError in your application, and using DISCARD to reset a failed transaction before retrying.
