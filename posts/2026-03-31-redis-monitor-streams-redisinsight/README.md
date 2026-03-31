# How to Monitor Redis Streams with RedisInsight

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, RedisInsight, Monitoring, Consumer Group

Description: Learn how to inspect Redis Stream entries, consumer groups, and pending messages using the RedisInsight browser for visual stream management.

---

Redis Streams are append-only data structures used for message queues, event logs, and real-time pipelines. RedisInsight provides a dedicated stream view that makes it easy to explore entries, inspect consumer groups, and investigate pending messages without writing complex `XREAD` or `XPENDING` commands.

## Viewing a Stream in RedisInsight

Navigate to the Browser tab and find a key with type `stream`. Click it to open the stream viewer. You see:

```text
Stream: events:orders
Total entries: 14,832
Last entry ID: 1700000000001-0
```

## Browsing Stream Entries

The stream viewer shows a paginated table of entries, newest first by default. Each row contains:

- Entry ID (timestamp + sequence)
- Fields and values

```text
ID                  | field         | value
1700000001234-0     | event_type    | order_placed
                    | order_id      | 99001
                    | amount        | 49.99
1700000001000-0     | event_type    | order_shipped
                    | order_id      | 98999
```

Switch between newest-first and oldest-first using the sort toggle.

## Filtering by Entry ID Range

Use the filter bar to view entries within a time window by entering entry IDs:

```text
From: 1699999999000-0
To:   1700000000000-0
```

This is equivalent to:

```text
XRANGE events:orders 1699999999000-0 1700000000000-0
```

## Viewing Consumer Groups

Click the "Consumer Groups" tab within the stream view. You see:

```text
Group Name       | Consumers | Pending | Last Delivered ID
orders-processor | 3         | 12      | 1700000000001-0
audit-logger     | 1         | 0       | 1700000000001-0
```

A non-zero pending count means messages have been delivered but not acknowledged.

## Inspecting Pending Messages

Click on a consumer group with pending messages to drill into them:

```text
Consumer         | Pending | Min Idle Time
worker-1         | 5       | 120s
worker-2         | 7       | 45s
worker-3         | 0       | -
```

High idle times indicate stuck or crashed consumers. Use the CLI to acknowledge or re-deliver those messages:

```text
> XACK events:orders orders-processor 1700000000001-0
(integer) 1

> XCLAIM events:orders orders-processor worker-3 60000 1700000000001-0
```

## Monitoring Stream Length

Watch the total entry count over time. In the Workbench, run:

```text
XLEN events:orders
```

If the stream grows unboundedly, set a max length:

```text
XADD events:orders MAXLEN ~ 100000 * event_type "order_placed" order_id "12345"
```

## Adding a Test Entry from RedisInsight

From the Browser, click "+" to add a new stream entry:

```text
Key: events:orders
ID: * (auto-generate)
Fields:
  event_type: test_event
  message: "hello from RedisInsight"
```

Click "Add Entry" to append it to the stream.

## Summary

RedisInsight provides a visual stream browser that simplifies inspecting entries, consumer groups, and pending messages. Use it to quickly identify stuck consumers, view entry ranges, and monitor stream growth - tasks that would otherwise require multiple complex CLI commands.
