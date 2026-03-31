# How to Use XGROUP DESTROY in Redis to Delete Consumer Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, Consumer Groups, Cleanup, Messaging

Description: Learn how to use XGROUP DESTROY to completely remove a consumer group from a Redis Stream, including all its consumers and pending message tracking.

---

## What Is XGROUP DESTROY?

`XGROUP DESTROY` removes an entire consumer group from a Redis Stream. This deletes the group's tracking state, all registered consumers, and all pending entries (PEL) associated with the group. The underlying stream and its messages are not affected - the data remains intact.

## Syntax

```text
XGROUP DESTROY key groupname
```

- `key` - the stream name
- `groupname` - the consumer group to destroy

Returns `1` if the group was destroyed, `0` if the group did not exist.

## Basic Usage

```bash
# Create a stream and a consumer group
XADD events * action "login" user "alice"
XGROUP CREATE events analytics 0

# Read messages to create pending entries
XREADGROUP GROUP analytics tracker COUNT 5 STREAMS events >

# Destroy the group (removes all consumers, PEL, and group tracking)
XGROUP DESTROY events analytics
# (integer) 1

# Attempting to destroy again returns 0
XGROUP DESTROY events analytics
# (integer) 0
```

## What Gets Deleted

When you call `XGROUP DESTROY`, the following are removed:

- The consumer group entry itself
- All consumers registered in the group (equivalent to calling `XGROUP DELCONSUMER` for each)
- All pending entries (PEL) - messages that were delivered but not yet acknowledged
- The group's last-delivered-id pointer

The stream itself and all messages within it remain untouched.

## Verifying the Group Is Gone

```bash
# Before destroy
XINFO GROUPS events
# Shows "analytics" group with consumer count and pending count

XGROUP DESTROY events analytics

# After destroy - group is not listed
XINFO GROUPS events
# (empty array)

# The stream messages are still there
XLEN events
# (integer) 1
```

## Use Cases for XGROUP DESTROY

- **Decommissioning a processing pipeline** - When retiring a feature or service, clean up the associated consumer group.
- **Resetting for re-processing** - Destroy and recreate the group starting at `0` to replay all messages.
- **Test teardown** - In integration tests, destroy groups between test runs to ensure clean state.

## Re-processing All Messages

```bash
# Destroy existing group to reset progress
XGROUP DESTROY events analytics

# Recreate starting from the very beginning of the stream
XGROUP CREATE events analytics 0

# Now XREADGROUP will deliver all messages from the start
XREADGROUP GROUP analytics tracker COUNT 100 STREAMS events >
```

## Application-Level Cleanup

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def teardown_consumer_group(stream: str, group: str):
    result = r.xgroup_destroy(stream, group)
    if result:
        print(f"Consumer group '{group}' on stream '{stream}' destroyed.")
    else:
        print(f"Consumer group '{group}' did not exist.")

def reset_consumer_group(stream: str, group: str, start_id: str = "0"):
    """Destroy and recreate a group to replay from a given position."""
    teardown_consumer_group(stream, group)
    r.xgroup_create(stream, group, id=start_id, mkstream=True)
    print(f"Consumer group '{group}' recreated starting from '{start_id}'.")

reset_consumer_group("events", "analytics", "0")
```

## Difference from XGROUP DELCONSUMER

| Operation | Scope | Use Case |
|-----------|-------|----------|
| `XGROUP DELCONSUMER` | Removes one consumer | Scaling down, consumer crash |
| `XGROUP DESTROY` | Removes entire group | Decommission, full reset |

## Summary

`XGROUP DESTROY` completely removes a consumer group and all its associated state (consumers and PEL) from a Redis Stream without affecting the stream data itself. It is the appropriate command when decommissioning a processing pipeline, resetting message processing to replay from the beginning, or cleaning up test environments. Always verify pending messages are handled before destroying a group in production.
