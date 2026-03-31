# How to Test Redis Cluster Resharding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Resharding, Testing, High Availability, Slots

Description: Test Redis Cluster resharding operations by validating data integrity, key slot migrations, and application availability during hash slot rebalancing.

---

## Understanding Resharding

Redis Cluster distributes 16384 hash slots across nodes. Resharding moves slots from one node to another. Testing resharding validates that:
- Keys are accessible before, during, and after migration
- Applications handle MOVED/ASK redirects transparently
- No data is lost during slot migration

## Setting Up a Test Cluster

Use the `create-cluster` utility:

```bash
# Create a 6-node cluster (3 primaries, 3 replicas)
for port in 7000 7001 7002 7003 7004 7005; do
    mkdir -p /tmp/redis-cluster/$port
    redis-server --port $port \
        --cluster-enabled yes \
        --cluster-config-file /tmp/redis-cluster/$port/nodes.conf \
        --cluster-node-timeout 5000 \
        --daemonize yes \
        --logfile /tmp/redis-cluster/$port/redis.log
done

# Initialize the cluster
redis-cli --cluster create \
    127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
    127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
    --cluster-replicas 1 --cluster-yes
```

## Pre-Resharding Data Validation

```bash
#!/bin/bash
# Write test data spread across multiple slots
redis-cli -c -p 7000 MSET \
    testkey:1 "value1" \
    testkey:2 "value2" \
    testkey:3 "value3" \
    testkey:4 "value4" \
    testkey:5 "value5"

# Verify all keys exist
for i in 1 2 3 4 5; do
    val=$(redis-cli -c -p 7000 GET testkey:$i)
    echo "testkey:$i = $val"
done

# Check cluster info
redis-cli -p 7000 CLUSTER INFO
redis-cli -p 7000 CLUSTER NODES
```

## Python - Continuous Reads During Resharding

```python
import redis
from redis.cluster import RedisCluster
import threading
import time

startup_nodes = [
    {"host": "127.0.0.1", "port": 7000},
    {"host": "127.0.0.1", "port": 7001},
    {"host": "127.0.0.1", "port": 7002},
]

rc = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)

# Write initial data
for i in range(100):
    rc.set(f"reshard:key:{i}", f"value:{i}")

errors = []
reads_ok = 0
stop_flag = threading.Event()

def continuous_reads():
    global reads_ok
    while not stop_flag.is_set():
        for i in range(100):
            try:
                val = rc.get(f"reshard:key:{i}")
                if val == f"value:{i}":
                    reads_ok += 1
                else:
                    errors.append(f"Wrong value for key {i}: {val}")
            except Exception as e:
                errors.append(str(e))
        time.sleep(0.1)

reader = threading.Thread(target=continuous_reads)
reader.start()

print("Start resharding now via redis-cli --cluster reshard ...")
print("Press Enter when resharding is complete.")
input()

stop_flag.set()
reader.join()

print(f"Successful reads: {reads_ok}")
print(f"Errors: {len(errors)}")
if errors:
    for e in errors[:5]:
        print(f"  {e}")
```

## Performing the Reshard

While the reader runs, execute the reshard:

```bash
# Identify node IDs
redis-cli -p 7000 CLUSTER NODES

# Move 1000 slots from node 7000 to 7001
redis-cli --cluster reshard 127.0.0.1:7000 \
    --cluster-from <node-id-7000> \
    --cluster-to <node-id-7001> \
    --cluster-slots 1000 \
    --cluster-yes
```

## Automated Slot Validation

```python
def validate_slot_distribution():
    nodes = rc.get_nodes()
    total_slots = 0
    for node in nodes:
        if node.server_type == "primary":
            slots = rc.cluster_slots()
            print(f"Node {node.host}:{node.port} - checking slot count...")

    info = rc.cluster_info()
    print(f"Cluster state: {info['cluster_state']}")
    print(f"Cluster slots assigned: {info['cluster_slots_assigned']}")
    assert info["cluster_state"] == "ok"
    assert int(info["cluster_slots_assigned"]) == 16384

validate_slot_distribution()
```

## Verifying Data Integrity After Reshard

```bash
#!/bin/bash
echo "Verifying data integrity after reshard..."
FAILED=0
for i in $(seq 1 100); do
    val=$(redis-cli -c -p 7000 GET "reshard:key:$i")
    expected="value:$i"
    if [ "$val" != "$expected" ]; then
        echo "FAIL: reshard:key:$i expected=$expected got=$val"
        FAILED=$((FAILED + 1))
    fi
done

if [ $FAILED -eq 0 ]; then
    echo "All keys verified successfully"
else
    echo "$FAILED keys failed verification"
    exit 1
fi
```

## Rebalancing All Slots Evenly

```bash
# Automatically rebalance slots across all nodes
redis-cli --cluster rebalance 127.0.0.1:7000 --cluster-use-empty-masters
```

Check the rebalance result:

```bash
redis-cli --cluster check 127.0.0.1:7000
```

## Cleanup

```bash
for port in 7000 7001 7002 7003 7004 7005; do
    redis-cli -p $port SHUTDOWN NOSAVE
done
rm -rf /tmp/redis-cluster
```

## Summary

Testing Redis Cluster resharding requires continuous read/write validation during slot migration to catch dropped keys or MOVED redirect errors. Cluster-aware clients handle MOVED and ASK redirections transparently, so most applications experience zero errors during resharding. Always verify slot counts sum to 16384 and that cluster state returns "ok" after resharding completes.
