# How to Disable Transparent Huge Pages for Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Linux, Performance, Configuration

Description: Disable Transparent Huge Pages (THP) on Linux to prevent Redis latency spikes and high copy-on-write memory usage during BGSAVE fork operations.

---

Transparent Huge Pages (THP) is a Linux kernel feature that automatically uses 2 MB memory pages instead of the standard 4 KB pages to reduce TLB pressure. While THP can help some workloads, it causes serious problems for Redis by increasing memory usage during fork-based snapshots and introducing latency spikes during page allocation.

## Why THP Hurts Redis

When Redis forks for BGSAVE or BGREWRITEAOF, copy-on-write (COW) copies pages that are written after the fork. With THP:

- A single 4 KB write triggers copying of the entire 2 MB huge page
- COW memory usage can be 500x larger than with standard pages
- Background compaction (khugepaged) causes random latency spikes

Redis explicitly warns about this:

```text
WARNING you have Transparent Huge Pages (THP) support enabled in your kernel.
This will create latency and memory usage issues with Redis.
To fix this issue run the command 'echo madvise > /sys/kernel/mm/transparent_hugepage/enabled'
```

## Checking Current THP Status

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
```

```text
[always] madvise never
```

`[always]` means THP is fully enabled. `[madvise]` means only enabled for processes that request it. `[never]` is what you want for Redis.

## Disabling THP Immediately

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

Verify:

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# always madvise [never]
```

## Making the Change Persistent

The `/sys` filesystem is ephemeral - changes are lost on reboot. Use one of these methods to persist:

### Via rc.local

```bash
sudo nano /etc/rc.local
```

Add before the `exit 0` line:

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### Via systemd service

```bash
sudo nano /etc/systemd/system/disable-thp.service
```

```text
[Unit]
Description=Disable Transparent Huge Pages
After=sysinit.target local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable disable-thp
sudo systemctl start disable-thp
```

## Container and Kubernetes Environments

In Kubernetes, use a privileged init container to disable THP at the node level before Redis starts:

```yaml
initContainers:
- name: disable-thp
  image: busybox
  securityContext:
    privileged: true
  command:
  - sh
  - -c
  - |
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
  volumeMounts:
  - name: sys
    mountPath: /sys
volumes:
- name: sys
  hostPath:
    path: /sys
```

## Measuring the Impact

Compare BGSAVE copy-on-write size before and after disabling THP:

```bash
redis-cli INFO persistence | grep rdb_last_cow_size
```

With THP enabled, a 2 GB Redis instance writing 1000 keys/sec during BGSAVE might show 500+ MB of COW. After disabling THP, the same workload typically shows under 50 MB.

Also measure latency:

```bash
redis-cli --latency -h localhost -p 6379
```

## Summary

Disable Transparent Huge Pages on any Linux host running Redis to prevent large copy-on-write memory usage during fork operations and latency spikes from background compaction. Set both `transparent_hugepage/enabled` and `transparent_hugepage/defrag` to `never`. Persist the change via a systemd service or init container in Kubernetes. Monitor `rdb_last_cow_size` in `INFO persistence` to confirm the improvement.
