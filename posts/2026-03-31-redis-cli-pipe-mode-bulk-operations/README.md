# How to Use Redis CLI in Pipe Mode for Bulk Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CLI, PIPE Mode, Bulk Operation, Performance

Description: Learn how to use redis-cli pipe mode to import large datasets, run bulk commands, and achieve maximum throughput using the Redis mass insertion protocol.

---

Redis CLI pipe mode (`--pipe`) is designed for bulk data insertion. It sends commands over the wire as fast as possible without waiting for individual replies, making it dramatically faster than running commands one at a time.

## When to Use Pipe Mode

Use `--pipe` when:
- Loading initial data into Redis (seed data, migrations)
- Importing CSV or JSON exports
- Generating thousands of keys for testing
- Running pre-computed batch operations

## Basic Usage

Pipe mode reads Redis protocol commands from stdin:

```bash
redis-cli --pipe < commands.txt
```

Where `commands.txt` contains Redis inline commands:

```text
SET user:1 "Alice"
SET user:2 "Bob"
SET user:3 "Carol"
EXPIRE user:1 3600
EXPIRE user:2 3600
```

## Generating Commands in Bash

Use a shell script to generate bulk commands and pipe them:

```bash
for i in $(seq 1 100000); do
  echo "SET key:$i value:$i"
done | redis-cli --pipe
```

Output:

```text
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 100000
```

## Generating Commands in Python

For more complex data:

```python
import subprocess

def generate_commands():
    for i in range(1, 100001):
        yield f"SET product:{i} '{{\"id\":{i},\"price\":{i * 0.99}}}'\n"
        yield f"EXPIRE product:{i} 86400\n"

proc = subprocess.Popen(
    ["redis-cli", "--pipe"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

for cmd in generate_commands():
    proc.stdin.write(cmd.encode())

proc.stdin.close()
stdout, stderr = proc.communicate()
print(stdout.decode())
```

## Using the Redis Protocol Format

For maximum performance, use the Redis Serialization Protocol (RESP) format directly. This avoids the overhead of parsing inline commands:

```bash
# Generate RESP format for "SET key1 value1"
printf "*3\r\n\$3\r\nSET\r\n\$4\r\nkey1\r\n\$6\r\nvalue1\r\n" | redis-cli --pipe
```

A helper script to generate RESP:

```bash
#!/bin/bash
redis_command() {
  local args=("$@")
  printf "*${#args[@]}\r\n"
  for arg in "${args[@]}"; do
    printf "\$${#arg}\r\n${arg}\r\n"
  done
}

for i in $(seq 1 1000); do
  redis_command SET "session:$i" "active"
done | redis-cli --pipe
```

## Comparing Performance

Single command loop (slow):

```bash
time for i in $(seq 1 10000); do
  redis-cli SET key:$i val:$i
done
# real: ~45s
```

Pipe mode (fast):

```bash
time for i in $(seq 1 10000); do
  echo "SET key:$i val:$i"
done | redis-cli --pipe
# real: ~0.3s
```

## Connecting to a Remote Redis

```bash
for i in $(seq 1 50000); do
  echo "SET key:$i value:$i"
done | redis-cli -h redis.example.com -p 6379 -a password --pipe
```

## Summary

Redis CLI pipe mode is the fastest way to import bulk data into Redis, sending commands without waiting for individual replies. Use it with generated command streams in bash or Python to achieve throughput orders of magnitude higher than running individual `redis-cli` commands in a loop.
