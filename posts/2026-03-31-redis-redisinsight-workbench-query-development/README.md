# How to Use RedisInsight Workbench for Query Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisInsight, Workbench, Query, Development

Description: Learn how to use RedisInsight Workbench to write, run, and save Redis commands with syntax highlighting, result visualization, and history tracking.

---

RedisInsight Workbench is an enhanced command editor designed for developing and testing Redis queries. Unlike the basic CLI panel, it supports multi-command scripts, result history, command saving, and visual result rendering for complex types like JSON and RediSearch results.

## Opening the Workbench

In RedisInsight, click the "Workbench" tab in the left sidebar (the `{}` icon). You see an editor pane on the left and results on the right.

## Writing Multi-Command Scripts

Unlike the CLI which runs one command at a time, the Workbench can run multiple commands together. Enter them on separate lines:

```text
SET product:100 '{"name":"Widget","price":9.99}'
GET product:100
TYPE product:100
TTL product:100
```

Click "Run All" or use `Ctrl+Enter` to execute all commands. Results appear on the right side, one block per command.

## Syntax Highlighting

The Workbench highlights command names in blue, keys in green, and string values in orange. Errors are highlighted in red before execution if the command is malformed.

## Running Only Selected Commands

Highlight specific commands and press `Ctrl+Enter` to run only those lines. This is useful when you have a large script but want to test individual parts.

## Viewing Results

For each command, results are shown with collapsible blocks:

```text
SET product:100     -> OK
GET product:100     -> '{"name":"Widget","price":9.99}'
TYPE product:100    -> string
TTL product:100     -> -1 (no expiry)
```

For hash results, RedisInsight renders them as a table. For JSON module results, it renders a formatted JSON tree.

## Using the Workbench for RediSearch

When you have the Search module installed, the Workbench renders search results visually:

```text
FT.CREATE idx:products ON JSON PREFIX 1 product:
  SCHEMA $.name AS name TEXT
         $.price AS price NUMERIC SORTABLE

FT.SEARCH idx:products "widget" RETURN 2 name price
```

Results show matched documents with highlighted terms.

## Saving Frequently Used Commands

Click the bookmark icon to save a command to your favorites. Access saved commands from the "My Queries" panel. This is useful for admin scripts you run regularly:

```text
# Saved: Check memory usage
MEMORY DOCTOR
INFO memory
```

## Command History

The Workbench keeps a history of previously executed scripts. Click the clock icon to browse past executions. Each entry shows the commands, timestamp, and results.

## Enabling Auto-Execution on Connect

For databases where you always run setup commands, save them in the Workbench and re-run from history after connecting.

## Workbench vs CLI

| Feature | CLI | Workbench |
|---------|-----|-----------|
| Multi-command | No | Yes |
| Save queries | No | Yes |
| Visual results | Basic | Rich |
| History | Session only | Persistent |
| Command docs | Inline | Inline |

## Summary

RedisInsight Workbench is the best environment for developing and testing Redis commands. Its multi-command execution, persistent history, visual result rendering, and query saving make it far more productive than the basic CLI for complex operations and repeatable admin tasks.
