# How to Use ClickHouse Client in Node.js Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Node.js, JavaScript, Client, Analytics

Description: Connect to ClickHouse from Node.js using the official client, run queries, and insert data with streaming support.

---

## Installation

```bash
npm install @clickhouse/client
```

The package supports both CommonJS and ESM. It works in Node.js 18+ and also has a browser-compatible variant.

## Creating a Client

```javascript
import { createClient } from '@clickhouse/client';

const client = createClient({
  url: 'http://localhost:8123',
  database: 'analytics',
  username: 'default',
  password: '',
  clickhouse_settings: {
    async_insert: 1,
    wait_for_async_insert: 0,
  },
});
```

## Running a Query

```javascript
const resultSet = await client.query({
  query: `SELECT event_name, count() AS cnt
          FROM events
          WHERE toDate(ts) >= today() - 7
          GROUP BY event_name
          ORDER BY cnt DESC
          LIMIT 10`,
  format: 'JSONEachRow',
});

const rows = await resultSet.json();
for (const row of rows) {
  console.log(`${row.event_name}: ${row.cnt}`);
}
```

## Inserting Rows

```javascript
await client.insert({
  table: 'events',
  values: [
    { user_id: 42, event_name: 'page_view', ts: new Date().toISOString() },
    { user_id: 43, event_name: 'click',     ts: new Date().toISOString() },
  ],
  format: 'JSONEachRow',
});
```

## Streaming Query Results

For large result sets, use the stream API to avoid buffering everything in memory.

```javascript
const stream = await client.query({
  query: 'SELECT * FROM events LIMIT 1000000',
  format: 'JSONEachRow',
});

for await (const rows of stream.stream()) {
  rows.forEach(row => process.stdout.write(row.text + '\n'));
}
```

## Parameterized Queries

```javascript
const resultSet = await client.query({
  query: `SELECT count() FROM events WHERE event_name = {name:String} AND user_id = {uid:UInt64}`,
  query_params: { name: 'purchase', uid: 42 },
  format: 'JSONEachRow',
});
```

Use ClickHouse's named parameter syntax `{param:Type}` to safely pass values.

## Closing the Connection

```javascript
await client.close();
```

In a long-running server, keep the client open and reuse it across requests.

## Express.js Integration

```javascript
import express from 'express';
const app = express();

app.get('/api/top-events', async (req, res) => {
  const rs = await client.query({
    query: 'SELECT event_name, count() FROM events GROUP BY event_name ORDER BY 2 DESC LIMIT 10',
    format: 'JSONEachRow',
  });
  res.json(await rs.json());
});

app.listen(3000);
```

## Summary

The official `@clickhouse/client` for Node.js supports queries, inserts, and streaming with a clean async/await API. Use named parameters for safe query composition and streaming for large result sets to keep memory usage bounded.
