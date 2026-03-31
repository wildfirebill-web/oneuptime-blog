# How Redis Handles Key Expiration in Replicas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Replication, Expiration

Description: Understand how key expiration works differently on Redis replicas vs primaries - why replicas can serve stale expired data and how to reason about consistency.

---

Key expiration on Redis replicas works differently from the primary. Replicas do not independently expire keys. Instead, they wait for expiration signals from the primary. This design has important implications for read consistency and capacity planning.

## Primary-Driven Expiration Model

Redis follows a primary-driven expiration model. When a key expires on the primary (via lazy or active expiration), the primary sends a `DEL` command to all replicas. Replicas delete the key upon receiving this command.

This means:
- Replicas never independently decide to delete a key
- Expired keys on replicas remain readable until the primary sends DEL
- The delay between expiration and DEL delivery depends on network latency and replication lag

## Why Replicas Can Serve Stale Data

If a key expires at time T:
1. Client reads the key from a replica at T+1
2. The primary has not yet lazy-expired the key (no read on primary)
3. Active expiration cycle has not yet picked up the key
4. The replica still returns the value

```bash
# On primary: key expires
redis-cli -h primary SET mykey "value" PX 5000
# After 5 seconds...

# On replica: key may still be returned
redis-cli -h replica GET mykey  # Might return "value"
```

## Checking Expiration on Replicas

You can check TTL on a replica and it will report the remaining time (negative if expired but not yet deleted):

```bash
redis-cli -h replica TTL mykey
# Returns: -2 if already deleted, or a positive number if still live
```

Note: TTL checks use the local clock on the replica and can return negative values for expired-but-not-yet-deleted keys.

## Replica Expiration Logic Change in Redis 3.2

Before Redis 3.2, replicas would always serve expired keys without filtering. Since Redis 3.2, replicas use their own clock to check if a key has logically expired when responding to client reads. Even if the DEL has not arrived from the primary yet, the replica will return nil for logically expired keys.

```bash
redis-cli INFO server | grep "redis_version"
```

## Implications for Read Workloads

If your application reads exclusively from replicas with `slave_serve_stale_data no`, it may receive errors for keys during replication lag:

```bash
redis-cli CONFIG GET slave-serve-stale-data
```

Setting `slave-serve-stale-data no` causes replicas to return errors when they are not in sync with the primary.

## Monitoring Expiration Propagation

Monitor expired_keys on both primary and replica:

```bash
redis-cli -h primary INFO stats | grep expired_keys
redis-cli -h replica INFO stats | grep expired_keys
```

If the replica's `expired_keys` count lags far behind the primary, replication is behind.

## Summary

Redis replicas do not expire keys independently. They wait for DEL propagation from the primary after the primary expires a key through lazy or active expiration. Since Redis 3.2, replicas use their local clock to return nil for logically expired keys, but physical deletion still depends on the primary sending DEL commands.
