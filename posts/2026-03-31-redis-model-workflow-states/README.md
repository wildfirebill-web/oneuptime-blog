# How to Model Workflow States in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Workflow, State Machine, Hash, Data Modeling

Description: Model workflow states and state machines in Redis using Hashes and Lua scripts for atomic transitions, with practical examples for order and approval flows.

---

Workflow state machines - order processing, approval queues, document publishing - need fast state reads, atomic transitions, and history tracking. Redis is well suited for this because it can store current state in a Hash, enforce transitions atomically with Lua scripts, and record history in a List or Stream.

## Data Layout

```text
workflow:{id}            -> Hash: current state, timestamps, metadata
workflow:{id}:history    -> List or Stream of state transitions
```

```bash
# Initialize a workflow
HSET workflow:order:5001 state "pending" created_at "2024-03-01T10:00:00Z" owner "user:1001"
```

## Defining Valid Transitions

Keep the transition map application-side and enforce it atomically in Redis using Lua:

```lua
-- transition.lua
local key = KEYS[1]
local from = ARGV[1]
local to   = ARGV[2]
local ts   = ARGV[3]

local current = redis.call('HGET', key, 'state')
if current ~= from then
  return redis.error_reply('INVALID_TRANSITION: current state is ' .. tostring(current))
end
redis.call('HSET', key, 'state', to, 'updated_at', ts)
redis.call('RPUSH', key .. ':history', from .. '->' .. to .. ' at ' .. ts)
return 'OK'
```

Load and call with EVALSHA for efficient reuse:

```bash
# Load the script
SCRIPT LOAD "$(cat transition.lua)"
# Returns a SHA, e.g. abc123

# Execute transition
EVALSHA abc123 1 workflow:order:5001 pending processing "2024-03-01T10:05:00Z"
```

## Python Example: Workflow Manager

```python
import redis
import datetime

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

TRANSITIONS = {
    "pending": ["processing", "cancelled"],
    "processing": ["shipped", "cancelled"],
    "shipped": ["delivered"],
    "cancelled": [],
    "delivered": [],
}

TRANSITION_SCRIPT = """
local key = KEYS[1]
local from_state = ARGV[1]
local to_state = ARGV[2]
local ts = ARGV[3]
local current = redis.call('HGET', key, 'state')
if current ~= from_state then
  return redis.error_reply('INVALID_TRANSITION')
end
redis.call('HSET', key, 'state', to_state, 'updated_at', ts)
redis.call('RPUSH', key .. ':history', from_state .. '->' .. to_state .. '@' .. ts)
return 'OK'
"""

sha = r.script_load(TRANSITION_SCRIPT)

def transition(workflow_id: str, to_state: str):
    key = f"workflow:{workflow_id}"
    current = r.hget(key, "state")
    if to_state not in TRANSITIONS.get(current, []):
        raise ValueError(f"Cannot transition from {current} to {to_state}")
    ts = datetime.datetime.utcnow().isoformat()
    r.evalsha(sha, 1, key, current, to_state, ts)

def get_history(workflow_id: str):
    return r.lrange(f"workflow:{workflow_id}:history", 0, -1)
```

## Using Redis Streams for History

For richer history with consumer group support, use a Stream instead of a List:

```bash
XADD workflow:order:5001:history * from pending to processing actor "user:1001"
XRANGE workflow:order:5001:history - +
```

## Expiring Completed Workflows

Archive completed workflows by setting an expiry after terminal states:

```bash
# After reaching "delivered" or "cancelled"
EXPIRE workflow:order:5001 604800   # 7 days
EXPIRE workflow:order:5001:history 604800
```

## Summary

Model workflow states in Redis with a Hash for current state and a List or Stream for history. Use Lua scripts to enforce atomic, validated state transitions that prevent race conditions. For long-lived workflows, set expiry on terminal states and use Streams when you need consumer group-based processing of transition events.
