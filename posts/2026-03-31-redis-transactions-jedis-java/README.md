# How to Use Redis Transactions with Jedis in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Jedis, Transaction, MULTI

Description: Learn how to use Redis transactions (MULTI/EXEC) with Jedis in Java, including optimistic locking with WATCH to prevent race conditions.

---

Redis transactions group multiple commands into an atomic block using `MULTI/EXEC`. All queued commands execute sequentially and atomically - no other client can interleave commands between them. Jedis exposes this through its `Transaction` class.

## Basic Transaction (MULTI/EXEC)

```java
import redis.clients.jedis.Transaction;
import redis.clients.jedis.Response;

try (Jedis jedis = pool.getResource()) {
    Transaction tx = jedis.multi();

    tx.set("account:1:balance", "500");
    tx.set("account:2:balance", "300");
    tx.incrBy("account:1:balance", -100);  // debit
    tx.incrBy("account:2:balance", 100);   // credit

    List<Object> results = tx.exec(); // atomic execution
    System.out.println("Transaction results: " + results);
}
```

## Discarding a Transaction

```java
try (Jedis jedis = pool.getResource()) {
    Transaction tx = jedis.multi();
    tx.set("key", "value");

    // Cancel before executing
    tx.discard();
    System.out.println("Transaction cancelled");
}
```

## Reading Results from a Transaction

```java
try (Jedis jedis = pool.getResource()) {
    Transaction tx = jedis.multi();

    Response<String> balance1 = tx.get("account:1:balance");
    Response<Long> newBalance = tx.incrBy("account:1:balance", 50);

    tx.exec();

    System.out.println("Old balance: " + balance1.get());
    System.out.println("New balance: " + newBalance.get());
}
```

## Optimistic Locking with WATCH

`WATCH` monitors keys before a transaction. If any watched key is modified by another client before `EXEC`, the transaction aborts (returns null).

```java
import redis.clients.jedis.Jedis;

public boolean transferFunds(JedisPool pool, String from, String to, long amount) {
    try (Jedis jedis = pool.getResource()) {
        int retries = 3;
        while (retries-- > 0) {
            jedis.watch(from, to);

            long fromBalance = Long.parseLong(jedis.get(from));
            if (fromBalance < amount) {
                jedis.unwatch();
                return false; // insufficient funds
            }

            Transaction tx = jedis.multi();
            tx.decrBy(from, amount);
            tx.incrBy(to, amount);

            List<Object> result = tx.exec();
            if (result != null) {
                return true; // success
            }
            // Retry if transaction was aborted
        }
    }
    return false;
}
```

## When EXEC Returns null

`exec()` returns `null` when a `WATCH`ed key was modified - indicating a conflict:

```java
try (Jedis jedis = pool.getResource()) {
    jedis.watch("inventory:item:42");

    String stock = jedis.get("inventory:item:42");

    Transaction tx = jedis.multi();
    tx.decrBy("inventory:item:42", 1);

    List<Object> result = tx.exec();
    if (result == null) {
        System.out.println("Conflict detected - retry");
    } else {
        System.out.println("Stock decremented successfully");
    }
}
```

## Error Handling in Transactions

Redis executes all commands even if one fails at runtime (e.g., wrong type). Syntax errors abort the whole transaction at queue time:

```java
try (Jedis jedis = pool.getResource()) {
    Transaction tx = jedis.multi();
    tx.set("str_key", "hello");
    tx.incr("str_key");  // will fail at runtime (wrong type)
    tx.set("other_key", "world");

    List<Object> results = tx.exec();
    // results[0] = "OK"
    // results[1] = JedisDataException (runtime error)
    // results[2] = "OK"
    // Other commands still execute!
}
```

## Summary

Redis transactions with Jedis use `jedis.multi()` to begin, queue commands on the `Transaction` object, and call `tx.exec()` to execute atomically. For race-condition-sensitive operations, use `jedis.watch(keys)` before opening the transaction - if any watched key changes before `exec()`, the transaction aborts and returns null. Implement a retry loop to handle optimistic locking conflicts.
