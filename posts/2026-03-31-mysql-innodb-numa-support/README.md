# How to Configure InnoDB NUMA Support in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Performance, NUMA, Configuration

Description: Learn how to configure InnoDB NUMA support in MySQL to optimize memory allocation on multi-socket servers and reduce cross-node memory latency.

---

## What Is NUMA and Why It Matters for MySQL

NUMA (Non-Uniform Memory Access) is a memory architecture used in multi-socket servers where each CPU socket has local memory that it can access faster than memory attached to other sockets. When MySQL runs on a NUMA machine without proper configuration, the OS may allocate memory from remote NUMA nodes, causing higher latency and degraded InnoDB buffer pool performance.

On a typical dual-socket server, memory access to a remote node can be 2-3x slower than local access. For a database server where InnoDB keeps hot data in the buffer pool, this matters significantly.

## Detecting NUMA Topology

Before configuring MySQL, verify your server's NUMA topology:

```bash
numactl --hardware
```

Sample output:

```text
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32767 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32768 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

## Configuring innodb_numa_interleave

MySQL provides the `innodb_numa_interleave` variable to enable NUMA memory interleaving for the InnoDB buffer pool. When enabled, the buffer pool pages are distributed across all NUMA nodes in a round-robin fashion, preventing all allocations from going to a single node.

Add this to your `my.cnf`:

```ini
[mysqld]
innodb_numa_interleave = ON
innodb_buffer_pool_size = 24G
```

Verify it is active at runtime:

```sql
SHOW VARIABLES LIKE 'innodb_numa_interleave';
```

```text
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_numa_interleave | ON    |
+------------------------+-------+
```

## Starting MySQL with numactl

An alternative approach - especially useful when `innodb_numa_interleave` is not available in older MySQL versions - is to launch `mysqld` using `numactl` with the interleave policy:

```bash
numactl --interleave=all /usr/sbin/mysqld --defaults-file=/etc/mysql/my.cnf &
```

For systemd-managed MySQL services, override the ExecStart in a drop-in file:

```bash
mkdir -p /etc/systemd/system/mysql.service.d
```

Create `/etc/systemd/system/mysql.service.d/numa.conf`:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --interleave=all /usr/sbin/mysqld
```

Then reload and restart:

```bash
systemctl daemon-reload
systemctl restart mysql
```

## Verifying NUMA Memory Distribution

After enabling NUMA interleaving, confirm that memory is distributed across nodes:

```bash
numastat -p $(pgrep mysqld)
```

You should see roughly equal `Numa_Hit` counts across nodes for the mysqld process. If all memory sits on one node, interleaving is not effective.

## Checking Buffer Pool Status

Monitor InnoDB buffer pool behavior after the change:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look at the `BUFFER POOL AND MEMORY` section for total allocated pages and ensure the buffer pool was fully initialized without allocation failures.

## When NUMA Interleaving Helps Most

NUMA interleaving is most beneficial when:
- The InnoDB buffer pool is large (greater than 50% of total RAM)
- The workload has a hot dataset accessed uniformly by all threads
- The server has 2 or more NUMA nodes

For workloads where a single NUMA node's memory is sufficient to hold the entire working set, binding MySQL to a single NUMA node with `numactl --cpunodebind=0 --membind=0` may perform better than interleaving.

## Summary

Configuring InnoDB NUMA support prevents performance degradation from remote memory accesses on multi-socket servers. Enable `innodb_numa_interleave = ON` in `my.cnf` or launch MySQL via `numactl --interleave=all` to distribute buffer pool memory evenly across NUMA nodes. Always verify with `numastat` that memory distribution is balanced after the change.
