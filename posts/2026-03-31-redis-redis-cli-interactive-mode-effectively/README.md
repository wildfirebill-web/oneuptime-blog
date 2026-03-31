# How to Use Redis CLI Interactive Mode Effectively

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis CLI, Developer Tools, Command Line, Productivity

Description: Master Redis CLI interactive mode with tips on navigation, history, multi-line input, pipelining, and scripting to boost your productivity.

---

## Starting Interactive Mode

Launch the Redis CLI in interactive mode by running `redis-cli` without any command arguments:

```bash
redis-cli -h 127.0.0.1 -p 6379
```

If your Redis instance requires authentication:

```bash
redis-cli -h 127.0.0.1 -p 6379 -a yourpassword
```

Once connected, you see the prompt:

```text
127.0.0.1:6379>
```

## Basic Navigation and Input

Interactive mode behaves like a standard REPL. You can type any Redis command and press Enter to execute it:

```text
127.0.0.1:6379> SET greeting "hello"
OK
127.0.0.1:6379> GET greeting
"hello"
```

Use the **Up/Down arrow keys** to navigate command history within the current session. History is also persisted to `~/.rediscli_history` across sessions.

## Switching Databases

Redis has 16 databases (0-15) by default. Use `SELECT` to switch:

```text
127.0.0.1:6379> SELECT 2
OK
127.0.0.1:6379[2]> SET temp "value"
OK
127.0.0.1:6379[2]> SELECT 0
OK
127.0.0.1:6379>
```

The prompt shows the current database number in brackets when it is not 0.

## Enabling Pretty Output

By default some commands return raw output. Enable pretty-printing for JSON-like structures using the `--resp3` flag for RESP3 protocol:

```bash
redis-cli --resp3
```

Or toggle output formatting inside interactive mode:

```text
127.0.0.1:6379> DEBUG JMAP
```

For HGETALL you can improve readability by using `CLIENT NO-EVICT` in combination with formatted commands:

```text
127.0.0.1:6379> HGETALL user:1001
1) "name"
2) "Alice"
3) "email"
4) "alice@example.com"
```

## Using MULTI/EXEC Transactions

Interactive mode supports multi-step transactions:

```text
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> INCR counter
QUEUED
127.0.0.1:6379> EXEC
1) (integer) 1
2) (integer) 2
```

If you want to abort a transaction, type `DISCARD`.

## Repeating Commands with Count

Run a command N times by prefixing it with a count in interactive mode (useful for benchmarking):

```text
127.0.0.1:6379> 10 INCR hits
(integer) 1
(integer) 2
...
(integer) 10
```

This executes `INCR hits` 10 times in sequence.

## Monitoring Real-Time Commands

Use `MONITOR` in interactive mode to watch all commands received by the server in real time:

```text
127.0.0.1:6379> MONITOR
OK
1711900000.123456 [0 127.0.0.1:52100] "SET" "greeting" "hello"
1711900001.234567 [0 127.0.0.1:52101] "GET" "greeting"
```

Press `Ctrl+C` to stop monitoring.

## Using the Built-In Help System

Type `HELP` followed by a command name for inline documentation:

```text
127.0.0.1:6379> HELP SET

  SET key value [NX | XX] [GET] [EX seconds | PX milliseconds |
  EXAT unix-time-seconds | PXAT unix-time-milliseconds | KEEPTTL]

  summary: Set the string value of a key
  since: 1.0.0
  group: string
```

Browse all commands in a group:

```text
127.0.0.1:6379> HELP @string
127.0.0.1:6379> HELP @hash
```

## Pipelining Commands from a File

Feed a file of commands into interactive mode using stdin:

```bash
cat commands.txt | redis-cli -h 127.0.0.1 -p 6379
```

Where `commands.txt` contains:

```text
SET key1 "value1"
SET key2 "value2"
GET key1
GET key2
```

For batched writes, use `--pipe` mode for maximum throughput:

```bash
redis-cli --pipe < commands.txt
```

## Executing a Single Command from Shell

Run a one-off command without entering interactive mode:

```bash
redis-cli -h 127.0.0.1 GET greeting
```

This is useful in shell scripts and CI pipelines.

## Connecting to Redis Over TLS

For TLS-enabled Redis:

```bash
redis-cli -h redis.example.com -p 6380 --tls \
  --cert /path/to/client.crt \
  --key /path/to/client.key \
  --cacert /path/to/ca.crt
```

## Useful Keyboard Shortcuts

| Shortcut      | Action                        |
|---------------|-------------------------------|
| `Ctrl+C`      | Interrupt current command     |
| `Ctrl+D`      | Exit interactive mode         |
| `Ctrl+L`      | Clear the screen              |
| `Up/Down`     | Navigate command history      |
| `Ctrl+R`      | Reverse search command history|
| `Tab`         | Auto-complete command names   |

## Summary

Redis CLI interactive mode is a powerful tool for day-to-day Redis operations. By mastering features like command repetition, inline help, the MONITOR command, transaction support, and TLS connectivity, you can inspect and manage Redis instances efficiently from the terminal. Combining interactive mode with piped file input and the `--pipe` flag enables high-throughput batch operations without a client library.
