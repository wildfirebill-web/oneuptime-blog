# How to Perform Parallel Object Operations with librados

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, librados, Parallel Processing, Performance

Description: Learn how to perform parallel object operations with librados using threads and async I/O to maximize throughput for bulk data processing workloads.

---

Parallel object operations with librados allow applications to saturate Ceph cluster bandwidth and IOPS by submitting many operations simultaneously across multiple pools, OSDs, and placement groups. This is essential for backup systems, data processing pipelines, and bulk migration tools.

## Parallelism Strategies

There are two main approaches to parallel operations in librados:

1. **Async I/O** - Submit multiple operations from a single thread using completion callbacks
2. **Multi-threading** - Run multiple threads each with their own IoCtx, each submitting operations

## Thread-Per-Worker Pattern (Python)

```python
import rados
import threading
import queue
import time

def worker(ioctx, task_queue, results):
    while True:
        task = task_queue.get()
        if task is None:
            break
        name, data = task
        try:
            ioctx.write_full(name, data)
            results.append({"name": name, "status": "ok"})
        except Exception as e:
            results.append({"name": name, "status": "error", "error": str(e)})
        finally:
            task_queue.task_done()

# Prepare work
NUM_WORKERS = 8
NUM_OBJECTS = 1000

with rados.Rados(conffile="/etc/ceph/ceph.conf") as cluster:
    # Each thread gets its own IoCtx
    ioctxs = [cluster.open_ioctx("mypool") for _ in range(NUM_WORKERS)]
    task_queue = queue.Queue(maxsize=NUM_WORKERS * 2)
    results = []

    # Start workers
    threads = []
    for i in range(NUM_WORKERS):
        t = threading.Thread(target=worker, args=(ioctxs[i], task_queue, results))
        t.start()
        threads.append(t)

    # Enqueue tasks
    start = time.time()
    for i in range(NUM_OBJECTS):
        task_queue.put((f"obj-{i}", f"data-{i}".encode() * 1024))

    # Signal workers to stop
    for _ in range(NUM_WORKERS):
        task_queue.put(None)

    for t in threads:
        t.join()

    elapsed = time.time() - start
    print(f"Wrote {NUM_OBJECTS} objects in {elapsed:.2f}s")
    print(f"Throughput: {NUM_OBJECTS/elapsed:.1f} ops/s")

    for ctx in ioctxs:
        ctx.close()
```

## Batch Async I/O Pattern

For even higher throughput, combine threading with async I/O:

```python
import rados
from concurrent.futures import ThreadPoolExecutor

def write_batch(cluster_conf, pool, objects):
    with rados.Rados(conffile=cluster_conf) as cluster:
        with cluster.open_ioctx(pool) as ioctx:
            completions = []
            for name, data in objects:
                comp = ioctx.aio_write_full(name, data)
                completions.append(comp)
            for comp in completions:
                comp.wait_for_complete_and_cb()
                comp.release()
    return len(objects)

BATCH_SIZE = 100
all_objects = [(f"file-{i}", b"x" * 4096) for i in range(10000)]
batches = [all_objects[i:i+BATCH_SIZE] for i in range(0, len(all_objects), BATCH_SIZE)]

with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [
        executor.submit(write_batch, "/etc/ceph/ceph.conf", "mypool", batch)
        for batch in batches
    ]
    total = sum(f.result() for f in futures)
    print(f"Total written: {total} objects")
```

## Parallel Reads for Bulk Export

```python
from concurrent.futures import ThreadPoolExecutor
import rados

def read_objects(conf, pool, names):
    results = {}
    with rados.Rados(conffile=conf) as cluster:
        with cluster.open_ioctx(pool) as ioctx:
            for name in names:
                results[name] = ioctx.read(name)
    return results

# Split object list into chunks
with rados.Rados(conffile="/etc/ceph/ceph.conf") as cluster:
    with cluster.open_ioctx("mypool") as ioctx:
        all_names = [obj.key for obj in ioctx.list_objects()]

chunks = [all_names[i:i+100] for i in range(0, len(all_names), 100)]

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [
        executor.submit(read_objects, "/etc/ceph/ceph.conf", "mypool", chunk)
        for chunk in chunks
    ]
    all_data = {}
    for f in futures:
        all_data.update(f.result())
```

## Summary

Parallel librados operations maximize Ceph cluster throughput by splitting workloads across multiple threads, each with its own IoCtx, and combining threading with async I/O for even greater pipeline depth. The thread-per-worker pattern suits general-purpose parallelism, while batch async I/O with `ThreadPoolExecutor` achieves the highest throughput for bulk writes and reads in data processing pipelines.
