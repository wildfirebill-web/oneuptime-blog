# How to Configure Redis Overcommit Memory on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Linux, Configuration, Performance

Description: Set vm.overcommit_memory=1 on Linux to prevent Redis BGSAVE failures caused by the kernel refusing memory allocation during fork operations.

---

When Redis performs a background save (BGSAVE) or AOF rewrite (BGREWRITEAOF), it forks a child process. On Linux with the default memory overcommit setting, the kernel may refuse to fork if there is not enough free memory for a full copy of the Redis process - even though copy-on-write means that memory is rarely fully needed. This causes save failures and log warnings.

## The Problem: Fork Failures Without Overcommit

With `vm.overcommit_memory=0` (the default), the kernel checks if enough physical memory is available before allowing a fork. Since Redis can use gigabytes of RAM, the check often fails even when the system has sufficient memory for practical use:

```text
# Redis log warning
WARNING overcommit_memory is set to 0! Background save may fail under low memory condition.
To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf
```

The BGSAVE may then fail:

```text
Background saving error
```

## Enabling Overcommit Memory

Set the sysctl value immediately:

```bash
sudo sysctl -w vm.overcommit_memory=1
```

Make it permanent across reboots:

```bash
echo 'vm.overcommit_memory=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Verify the setting:

```bash
sysctl vm.overcommit_memory
# vm.overcommit_memory = 1
```

## Understanding the Three Overcommit Modes

| Value | Behavior |
| --- | --- |
| 0 (default) | Kernel estimates available memory; may refuse allocation even if swap is available |
| 1 | Always allow memory allocation; rely on OOM killer if memory runs out |
| 2 | Never allocate more than swap + a percentage of physical RAM |

For Redis, `1` is the correct setting. It allows the fork to proceed, relying on copy-on-write semantics to keep actual memory usage low during the snapshot.

## How Copy-on-Write Makes This Safe

When Redis forks for BGSAVE:

- The child and parent share the same memory pages initially (zero copy)
- Only pages that are written to after the fork are duplicated
- If the workload is mostly reads during the save window, memory usage barely increases

With overcommit enabled, the kernel allows the fork based on virtual memory rather than physical memory, which matches how Redis actually uses memory after fork.

## Monitoring Fork Memory Usage

Check how much memory the fork actually used:

```bash
redis-cli INFO persistence | grep rdb_last_cow_size
```

```text
rdb_last_cow_size:15728640
```

This shows the copy-on-write overhead in bytes (about 15 MB in this example). If this value is high relative to your dataset, many writes are occurring during saves - consider reducing save frequency or write rate.

## Container and Kubernetes Considerations

In Kubernetes, you cannot change kernel sysctl values from within a container by default. You need a privileged init container or a node-level configuration:

```yaml
# In a DaemonSet or as a node initialization script
initContainers:
- name: sysctl-tuner
  image: busybox
  securityContext:
    privileged: true
  command: ['sh', '-c', 'sysctl -w vm.overcommit_memory=1']
```

Or configure it as a node `sysctl` in your node pool settings.

## Combining With maxmemory

Even with overcommit enabled, always set `maxmemory` to prevent Redis from consuming all available RAM:

```text
# redis.conf
maxmemory 6gb
maxmemory-policy allkeys-lru
```

This ensures Redis proactively evicts keys before reaching system memory limits.

## Summary

Set `vm.overcommit_memory=1` on any Linux host running Redis to prevent BGSAVE and BGREWRITEAOF failures caused by conservative kernel memory allocation checks. The setting is safe for Redis because forked child processes use copy-on-write semantics and rarely need the full memory allocation they request. Persist the setting in `/etc/sysctl.conf` and handle it via privileged init containers in Kubernetes environments.
