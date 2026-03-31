# How to Use XGROUP CREATECONSUMER in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, XGROUP, Consumer Group, Consumer

Description: Learn how to use XGROUP CREATECONSUMER to explicitly pre-register a consumer in a Redis Stream consumer group before it starts reading messages.

---

Redis normally creates consumers automatically when they first call `XREADGROUP`. `XGROUP CREATECONSUMER` lets you pre-register a consumer explicitly - useful for setup scripts, capacity planning, or ensuring a consumer exists before your application starts.

## How XGROUP CREATECONSUMER Works

`XGROUP CREATECONSUMER` adds a consumer entry to an existing consumer group without delivering any messages. The consumer starts with zero pending messages. It will appear in `XINFO CONSUMERS` output immediately, even before it has read anything.

## Syntax

```redis
XGROUP CREATECONSUMER key groupname consumername
```

- `key` - stream name
- `groupname` - consumer group the consumer belongs to
- `consumername` - name for the new consumer

Returns `1` if the consumer was created, `0` if it already existed.

## Prerequisites

The consumer group must already exist. Create it first if needed:

```redis
XGROUP CREATE mystream workers $ MKSTREAM
```

## Examples

### Pre-register a Consumer

```redis
XGROUP CREATECONSUMER mystream workers worker-1
```

### Verify the Consumer Exists

```redis
XINFO CONSUMERS mystream workers
```

Example output before any messages are read:

```text
1) 1) "name"
   2) "worker-1"
   3) "pending"
   4) (integer) 0
   5) "idle"
   6) (integer) 0
```

### Pre-register Multiple Consumers

Set up all consumers for a known worker pool before starting them:

```redis
XGROUP CREATECONSUMER mystream workers worker-1
XGROUP CREATECONSUMER mystream workers worker-2
XGROUP CREATECONSUMER mystream workers worker-3
```

### Idempotent Consumer Setup

`XGROUP CREATECONSUMER` returns `0` (not an error) if the consumer already exists, making it safe to call in startup scripts:

```bash
redis-cli XGROUP CREATECONSUMER mystream workers "$HOSTNAME"
# Returns 0 if already registered, 1 if newly created - both are fine
```

## Automatic vs Explicit Creation

| Method | When Created | Use Case |
|---|---|---|
| Automatic (via XREADGROUP) | First read | Simple applications |
| Explicit (XGROUP CREATECONSUMER) | Pre-registration | Infrastructure-as-code, startup scripts |

## Use Cases

- **Infrastructure provisioning** - register all consumers as part of deployment scripts
- **Consumer name validation** - ensure only known consumer names are used in a group
- **Monitoring setup** - pre-create consumers so dashboards show them even before they connect
- **Kubernetes pod startup** - register the pod's consumer name during init before the main loop starts

## Summary

`XGROUP CREATECONSUMER` fills the gap between group creation and first message consumption by allowing explicit consumer pre-registration. It is idempotent (safe to call multiple times) and integrates cleanly into deployment pipelines and startup scripts. In most applications, consumers are created automatically via `XREADGROUP`, but explicit creation gives you finer control over group membership.
