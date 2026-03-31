# How to Configure Redis Sentinel Quorum

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sentinel, Quorum, Configuration, High Availability

Description: Learn how Redis Sentinel quorum works, how to set the right quorum value, and why it matters for preventing false failovers and split-brain.

---

Quorum is the minimum number of Sentinel processes that must agree a primary is down before initiating a failover. Getting this value right is critical - too low causes false failovers, too high makes your system resistant to legitimate failures.

## How Quorum Works

Quorum has two roles in Redis Sentinel:

1. **Failure detection**: At least `quorum` Sentinels must mark the primary as S-DOWN (subjectively down) to declare O-DOWN (objectively down)
2. **Failover authorization**: A majority of Sentinels must vote for the Sentinel that will perform the failover

Note: quorum and majority are different. Quorum controls detection; majority controls authorization.

```text
With 3 Sentinels:
  - quorum=2: 2 must agree primary is down (detection)
  - majority=2: 2 must vote for failover leader (authorization)
```

## Setting Quorum

```bash
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
#                                            ^ quorum
```

The quorum value is the last parameter of `sentinel monitor`.

Common quorum configurations:

```text
3 Sentinels -> quorum=2 (majority, recommended)
5 Sentinels -> quorum=3 (majority, recommended)
5 Sentinels -> quorum=2 (faster detection, less safe)
```

## Quorum vs Majority

```bash
# Check how many Sentinels are known
redis-cli -p 26379 SENTINEL masters
# Look for "num-other-sentinels": should be (total - 1)
```

For failover to succeed, you need:
- At least `quorum` Sentinels to agree on O-DOWN
- A majority of all known Sentinels to elect the failover leader

With 3 Sentinels and quorum=2:
- 2 Sentinels agree on O-DOWN -> failover authorized
- 2 out of 3 vote for leader -> majority achieved
- Works even if 1 Sentinel is down

With 3 Sentinels and quorum=1:
- 1 Sentinel sees primary down -> O-DOWN declared
- But still needs 2/3 to elect a leader
- Can detect failures faster but may false-positive on network glitches

## Changing Quorum at Runtime

```bash
redis-cli -p 26379 SENTINEL SET mymaster quorum 3
```

Verify:

```bash
redis-cli -p 26379 SENTINEL masters
# Look for "quorum" field
```

## Quorum for Multiple Monitored Masters

Each monitored master has its own quorum:

```bash
# sentinel.conf
sentinel monitor master1 192.168.1.10 6379 2
sentinel monitor master2 192.168.1.20 6379 2
sentinel monitor critical-master 192.168.1.30 6379 3
```

You can set a higher quorum for more critical services.

## What Happens with a Split Quorum

If fewer than `quorum` Sentinels can communicate, no failover occurs - this is intentional:

```text
Scenario: Network splits 3 Sentinels into [S1] and [S2, S3]
quorum=2

Partition A: S1 sees primary is down
  -> Only 1 sentinel agrees, cannot reach quorum=2
  -> No failover (correct, avoids split-brain)

Partition B: S2 + S3 see primary is down
  -> 2 sentinels agree, quorum reached
  -> Failover proceeds (correct)
```

## Summary

Redis Sentinel quorum is the minimum number of Sentinels that must agree a primary is unreachable before triggering failover. Set it to a majority of your Sentinels (e.g., 2 out of 3, 3 out of 5) to prevent false failovers from transient network issues while ensuring genuine failures are detected reliably.
