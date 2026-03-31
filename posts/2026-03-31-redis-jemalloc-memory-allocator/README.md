# How Redis jemalloc Memory Allocator Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memory, jemalloc, Internals, Performance

Description: Learn how Redis uses jemalloc to manage heap memory, reduce fragmentation, and why memory usage reported by Redis can differ from OS-level readings.

---

Redis ships with jemalloc as its default memory allocator on Linux. Understanding how jemalloc works explains why Redis memory reports sometimes surprise operators and how to tune allocation behavior.

## Why Not the System malloc?

The system `glibc` malloc is a general-purpose allocator tuned for many workloads but prone to fragmentation under Redis-like allocation patterns (many small, variable-length allocations with frequent frees). jemalloc was designed specifically to reduce fragmentation in long-running server processes.

## How jemalloc Organizes Memory

jemalloc divides memory into three allocation size classes:

```text
Small  (<= 8 bytes up to ~14KB)  - thread-local caches (tcache)
Large  (14KB to 4MB)             - directly from arena
Huge   (> 4MB)                   - mmap'd directly
```

Each arena has its own set of size-class bins with free lists. Redis uses multiple arenas to reduce contention when IO threads are enabled.

## Checking jemalloc in Your Redis Build

```bash
redis-server --version
# Redis server v=7.2.0 ... allocator=jemalloc-5.3.0
```

```bash
redis-cli INFO memory | grep allocator
# mem_allocator:jemalloc-5.3.0
```

## Understanding the Memory Numbers

```bash
redis-cli INFO memory
```

Key fields:

```text
used_memory:           bytes Redis thinks it has allocated (via jemalloc tracking)
used_memory_rss:       bytes the OS reports for the Redis process (RSS)
mem_fragmentation_ratio: used_memory_rss / used_memory
```

A fragmentation ratio above 1.5 means jemalloc is holding significant unused memory in its internal bins that the OS still counts as resident.

## Active Defragmentation

Redis 4.0+ can ask jemalloc to move live objects to freshly allocated memory, releasing fragmented pages:

```text
# redis.conf
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
```

jemalloc exposes `JEMALLOC_PURGE` to return free pages to the OS. Redis calls this during `serverCron`.

## Manually Purging Memory

Force Redis to return free jemalloc memory to the OS immediately:

```bash
redis-cli MEMORY PURGE
```

This calls `je_mallopt(M_PURGE, 0)` internally, which tells jemalloc to release all cached free pages.

## Profiling Allocations

Enable jemalloc heap profiling (requires a debug build):

```bash
MALLOC_CONF=prof:true,prof_prefix:/tmp/redis.heap redis-server
```

Then dump a snapshot:

```bash
redis-cli MEMORY MALLOC-STATS
```

## Comparing Allocators

You can compile Redis with alternative allocators:

```bash
# libc malloc
make MALLOC=libc

# tcmalloc
make MALLOC=tcmalloc
```

Run benchmarks with `redis-benchmark` and compare `used_memory_rss` over time to see fragmentation differences.

## Summary

Redis uses jemalloc to minimize heap fragmentation through size-class bins, per-arena allocation, and thread-local caches. The gap between `used_memory` and `used_memory_rss` reflects jemalloc's internal free lists. Use `MEMORY PURGE` and active defragmentation to reclaim fragmented memory during low-traffic periods.
