# How to Test Redis Failover Scenarios

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Failover, Sentinel, High Availability, Testing, Disaster Recovery

Description: Test Redis failover scenarios using Redis Sentinel and Docker Compose to validate that your application handles primary failures, reconnections, and data consistency correctly.

---

## Why Test Failover

Failover behavior is only reliable if you test it before an incident occurs. Testing validates that:
- Your application reconnects to the new primary automatically
- In-flight operations are handled gracefully
- Write latency during failover is within acceptable bounds
- No data is silently lost

## Setting Up a Test Environment

Use Docker Compose to create a Sentinel-based HA cluster:

```yaml
version: "3.9"
services:
  redis-primary:
    image: redis:7-alpine
    command: redis-server --port 6379
    ports:
      - "6379:6379"

  redis-replica:
    image: redis:7-alpine
    command: redis-server --port 6379 --replicaof redis-primary 6379
    ports:
      - "6380:6379"

  sentinel-1:
    image: redis:7-alpine
    command: >
      sh -c 'echo "sentinel monitor mymaster redis-primary 6379 2
      sentinel down-after-milliseconds mymaster 5000
      sentinel failover-timeout mymaster 10000
      sentinel parallel-syncs mymaster 1" > /tmp/sentinel.conf &&
      redis-server /tmp/sentinel.conf --sentinel --port 26379'
    ports:
      - "26379:26379"
    depends_on:
      - redis-primary
      - redis-replica

  sentinel-2:
    image: redis:7-alpine
    command: >
      sh -c 'echo "sentinel monitor mymaster redis-primary 6379 2
      sentinel down-after-milliseconds mymaster 5000
      sentinel failover-timeout mymaster 10000
      sentinel parallel-syncs mymaster 1" > /tmp/sentinel.conf &&
      redis-server /tmp/sentinel.conf --sentinel --port 26380'
    ports:
      - "26380:26380"
    depends_on:
      - redis-primary
      - redis-replica
```

Start the cluster:

```bash
docker compose up -d
```

## Triggering a Manual Failover

Test failover by sending the FAILOVER command via Sentinel:

```bash
redis-cli -p 26379 sentinel failover mymaster
```

Or by stopping the primary container:

```bash
docker compose stop redis-primary
```

## Python Application Failover Test

```python
import redis
import time
import threading
import subprocess

def test_failover_transparency():
    sentinel = redis.Sentinel(
        [("localhost", 26379), ("localhost", 26380)],
        socket_timeout=0.5
    )

    errors = []
    successful_writes = 0
    stop_flag = threading.Event()

    def continuous_writes():
        nonlocal successful_writes
        master = sentinel.master_for("mymaster", socket_timeout=1, decode_responses=True)
        while not stop_flag.is_set():
            try:
                master.set("test:counter", str(successful_writes))
                successful_writes += 1
                time.sleep(0.1)
            except redis.ConnectionError as e:
                errors.append(str(e))
                # Re-discover primary after failover
                try:
                    master = sentinel.master_for("mymaster", socket_timeout=2, decode_responses=True)
                except Exception:
                    pass

    writer = threading.Thread(target=continuous_writes)
    writer.start()

    # Let writes run for 3 seconds
    time.sleep(3)

    print(f"Triggering failover...")
    subprocess.run(["redis-cli", "-p", "26379", "sentinel", "failover", "mymaster"])

    # Run for 10 more seconds after failover
    time.sleep(10)
    stop_flag.set()
    writer.join()

    print(f"Successful writes: {successful_writes}")
    print(f"Errors during failover: {len(errors)}")
    assert successful_writes > 0
    print("Failover test passed - application recovered automatically")

test_failover_transparency()
```

## Testing Read Replica Fallback

```python
def test_read_replica_failover():
    sentinel = redis.Sentinel(
        [("localhost", 26379)],
        socket_timeout=0.5
    )

    # Write to master
    master = sentinel.master_for("mymaster", decode_responses=True)
    master.set("read:test", "value_from_master")
    time.sleep(0.5)

    # Read from replica
    replica = sentinel.slave_for("mymaster", decode_responses=True)
    val = replica.get("read:test")
    assert val == "value_from_master", f"Expected value_from_master, got {val}"
    print("Read replica is synced correctly")

test_read_replica_failover()
```

## Checking Data Consistency After Failover

```bash
# Before failover - write to primary
redis-cli -p 6379 SET pre_failover_key "exists"

# Trigger failover
redis-cli -p 26379 SENTINEL FAILOVER mymaster

# Wait for failover to complete
sleep 10

# Check which instance is now primary
redis-cli -p 26379 SENTINEL GET-MASTER-ADDR-BY-NAME mymaster

# Verify data survived
redis-cli -p 6379 GET pre_failover_key
```

## Measuring Failover Duration

```python
def measure_failover_duration():
    sentinel = redis.Sentinel([("localhost", 26379)])
    master = sentinel.master_for("mymaster", socket_timeout=1)

    failover_start = None
    recovered_at = None

    subprocess.Popen(["redis-cli", "-p", "26379", "sentinel", "failover", "mymaster"])
    failover_start = time.time()

    for _ in range(60):
        try:
            master.ping()
            recovered_at = time.time()
            break
        except (redis.ConnectionError, redis.TimeoutError):
            master = sentinel.master_for("mymaster", socket_timeout=1)
            time.sleep(0.5)

    if recovered_at:
        print(f"Failover completed in {recovered_at - failover_start:.2f}s")
    else:
        print("Failover did not complete within 30s")

measure_failover_duration()
```

## Summary

Testing Redis failover requires a Sentinel-based test environment, automated failover triggers, and application-level monitoring for reconnection behavior. Applications should use a Sentinel-aware client that automatically re-discovers the new primary after failover. Measuring failover duration and tracking write errors during the transition window validates that your HA setup meets your availability requirements.
