# How to Use ClickHouse with Next.js API Routes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Next.js, Analytics, API Route, TypeScript, Node.js

Description: Connect ClickHouse to Next.js API Routes to power server-side analytics queries without a separate backend service.

---

Next.js API Routes let you run server-side code directly inside your frontend project. Pairing them with ClickHouse gives you a lightweight path to real-time analytics without deploying a separate backend.

## Install the ClickHouse Client

```bash
npm install @clickhouse/client
```

## Create a Shared ClickHouse Client

Create a singleton so the connection is not re-created on every request:

```typescript
// lib/clickhouse.ts
import { createClient } from '@clickhouse/client';

const clickhouse = createClient({
  host: process.env.CLICKHOUSE_HOST ?? 'http://localhost:8123',
  username: process.env.CLICKHOUSE_USER ?? 'default',
  password: process.env.CLICKHOUSE_PASSWORD ?? '',
  database: process.env.CLICKHOUSE_DB ?? 'default',
});

export default clickhouse;
```

## Create the Events Table

```sql
CREATE TABLE IF NOT EXISTS page_events (
    path       String,
    referrer   String,
    user_agent String,
    ip         String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY created_at;
```

## Write an API Route to Track Events

```typescript
// pages/api/track.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import clickhouse from '../../lib/clickhouse';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { path } = req.body as { path: string };
  const referrer = req.headers.referer ?? '';
  const userAgent = req.headers['user-agent'] ?? '';
  const ip = req.socket.remoteAddress ?? '';

  await clickhouse.insert({
    table: 'page_events',
    values: [{ path, referrer, user_agent: userAgent, ip }],
    format: 'JSONEachRow',
  });

  res.status(200).json({ ok: true });
}
```

## Write an API Route to Query Analytics

```typescript
// pages/api/analytics.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import clickhouse from '../../lib/clickhouse';

export default async function handler(
  _req: NextApiRequest,
  res: NextApiResponse
) {
  const result = await clickhouse.query({
    query: `
      SELECT
        path,
        count() AS views,
        uniqExact(ip) AS unique_visitors
      FROM page_events
      WHERE created_at >= now() - INTERVAL 7 DAY
      GROUP BY path
      ORDER BY views DESC
      LIMIT 20
    `,
    format: 'JSONEachRow',
  });

  const data = await result.json();
  res.status(200).json(data);
}
```

## Display Analytics in a React Page

```typescript
// pages/dashboard.tsx
import { useEffect, useState } from 'react';

type Row = { path: string; views: number; unique_visitors: number };

export default function Dashboard() {
  const [rows, setRows] = useState<Row[]>([]);

  useEffect(() => {
    fetch('/api/analytics')
      .then((r) => r.json())
      .then(setRows);
  }, []);

  return (
    <table>
      <thead>
        <tr><th>Path</th><th>Views</th><th>Unique</th></tr>
      </thead>
      <tbody>
        {rows.map((r) => (
          <tr key={r.path}>
            <td>{r.path}</td>
            <td>{r.views}</td>
            <td>{r.unique_visitors}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

## Environment Variables

```text
CLICKHOUSE_HOST=http://localhost:8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=secret
CLICKHOUSE_DB=analytics
```

## Summary

Next.js API Routes provide a simple integration point for ClickHouse. A shared singleton client avoids connection overhead, and the combination of a tracking endpoint and a query endpoint gives you a complete lightweight analytics backend within a single Next.js project.
