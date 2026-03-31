# How to Use ACL CAT in Redis to List Command Categories

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ACL, Security, Commands, Access Control

Description: Learn how to use ACL CAT in Redis to list available command categories and the commands within each category, essential for building precise ACL rules.

---

## What Is ACL CAT

Redis Access Control Lists (ACLs) let you restrict which commands users can run. Commands are organized into categories like `read`, `write`, `dangerous`, and `string`. `ACL CAT` lets you explore these categories and their member commands, which is essential when defining granular user permissions.

```text
ACL CAT [categoryname]
```

- Without arguments: lists all available category names
- With a category name: lists all commands in that category

## List All Categories

```bash
ACL CAT
```

Returns all command categories:

```text
 1) "keyspace"
 2) "read"
 3) "write"
 4) "set"
 5) "sortedset"
 6) "list"
 7) "hash"
 8) "string"
 9) "bitmap"
10) "hyperloglog"
11) "geo"
12) "stream"
13) "pubsub"
14) "admin"
15) "fast"
16) "slow"
17) "blocking"
18) "dangerous"
19) "connection"
20) "transaction"
21) "scripting"
```

## List Commands in a Category

```bash
ACL CAT read
```

```text
 1) "geodist"
 2) "georadiusbymember_ro"
 3) "get"
 4) "getbit"
 5) "getrange"
 6) "hget"
 7) "hgetall"
 8) "hmget"
 ...
```

```bash
ACL CAT write
ACL CAT dangerous
ACL CAT pubsub
ACL CAT admin
```

## Understanding Key Categories

| Category | Description |
|----------|-------------|
| `read` | Commands that only read data |
| `write` | Commands that modify data |
| `dangerous` | Commands with potential for data loss or exposure |
| `admin` | Server administration commands |
| `pubsub` | Pub/Sub commands |
| `scripting` | Lua/Functions commands |
| `fast` | O(1) or similar fast commands |
| `slow` | Commands that may be slow |

## Practical Example in Python

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

# List all categories
categories = client.acl_cat()
print("Available ACL categories:")
for cat in sorted(categories):
    print(f"  @{cat}")

print()

# List commands in specific categories
for category in ['read', 'write', 'dangerous', 'admin']:
    commands = client.acl_cat(category)
    print(f"@{category} ({len(commands)} commands):")
    print(f"  {', '.join(sorted(commands)[:5])}...")  # Show first 5
    print()
```

## Practical Example in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient();
await client.connect();

// List all categories
const categories = await client.aclCat();
console.log('All categories:');
categories.forEach(cat => console.log(`  @${cat}`));

// List commands in the dangerous category
const dangerousCommands = await client.aclCat('dangerous');
console.log('\nDangerous commands:');
dangerousCommands.forEach(cmd => console.log(`  ${cmd}`));
```

## Using ACL CAT to Build User Rules

After exploring categories, use them in ACL rules. The `@` prefix references a category:

```bash
# Allow read-only access
ACL SETUSER readonly on >password ~* &* +@read

# Allow string and hash operations
ACL SETUSER app_user on >secret ~app:* &* +@string +@hash

# Allow everything except dangerous commands
ACL SETUSER safe_admin on >password ~* &* +@all -@dangerous

# Allow pubsub only
ACL SETUSER pubsub_consumer on >pass ~* &* +@pubsub
```

## Practical ACL Rule Building Workflow

```python
import redis

client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def build_acl_rule(username, password, key_patterns, categories=None, commands=None):
    """Build and apply an ACL rule using category exploration."""

    # Explore categories first
    all_cats = client.acl_cat()
    print(f"Available categories: {', '.join(all_cats)}")

    rule_parts = [f"ACL SETUSER {username} on >{password}"]

    # Add key patterns
    for pattern in key_patterns:
        rule_parts.append(f"~{pattern}")

    # Add channel access
    rule_parts.append("&*")

    # Add category permissions
    if categories:
        for cat in categories:
            if cat in all_cats:
                rule_parts.append(f"+@{cat}")
            else:
                print(f"Warning: category '@{cat}' not found")

    # Add specific command permissions
    if commands:
        for cmd in commands:
            rule_parts.append(f"+{cmd}")

    rule = " ".join(rule_parts)
    print(f"\nACL rule: {rule}")
    return rule

build_acl_rule(
    username="api_service",
    password="strongpassword",
    key_patterns=["api:*", "session:*"],
    categories=["read", "write", "string", "hash"],
    commands=["expire", "ttl"]
)
```

## Exploring the Dangerous Category

```bash
ACL CAT dangerous
```

Common dangerous commands include:
- `FLUSHDB`, `FLUSHALL` - delete all data
- `CONFIG` - change server configuration
- `DEBUG` - server debug commands
- `REPLICAOF` - change replication topology
- `KEYS` - blocks server during scan

```bash
# Deny all dangerous commands explicitly
ACL SETUSER myuser on >pass ~* &* +@all -@dangerous
```

## Scripting Category

```bash
ACL CAT scripting
# eval, evalsha, fcall, function, script, ...
```

```bash
# Allow safe scripting but not flush
ACL SETUSER scripter on >pass ~* &* +@scripting -script|flush
```

## Summary

`ACL CAT` is your catalog for building precise Redis ACL rules - it lists all command categories and the specific commands in each. Use it to understand which commands fall under `@dangerous`, `@read`, `@write`, and other categories so you can grant least-privilege access to Redis users in production environments.
