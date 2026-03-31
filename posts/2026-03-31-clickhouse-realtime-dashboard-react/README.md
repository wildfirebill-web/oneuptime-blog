# How to Build Real-Time Dashboards with ClickHouse and React

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, React, Dashboard, Real-Time, Analytics

Description: Build a real-time analytics dashboard in React that polls a ClickHouse-backed API and renders charts using Recharts.

---

## Architecture

```text
React (polling every 30s) --> Node.js API --> ClickHouse
```

The React frontend polls an API endpoint. The Node.js backend queries ClickHouse and returns aggregated data. This is simpler than WebSockets and sufficient for most dashboard use cases.

## Backend API (Express + ClickHouse)

```typescript
import express from 'express';
import { createClient } from '@clickhouse/client';

const app = express();
const ch = createClient({ url: 'http://localhost:8123', database: 'analytics' });

app.get('/api/dashboard', async (req, res) => {
  const rs = await ch.query({
    query: `SELECT toStartOfMinute(ts) AS minute,
                   count()             AS events,
                   countIf(error = 1)  AS errors
            FROM api_logs
            WHERE ts >= now() - INTERVAL 60 MINUTE
            GROUP BY minute
            ORDER BY minute`,
    format: 'JSONEachRow',
  });
  res.json(await rs.json());
});

app.listen(4000);
```

## React Component with Polling

```tsx
import { useEffect, useState, useCallback } from 'react';
import { LineChart, Line, XAxis, YAxis, Tooltip, Legend, ResponsiveContainer } from 'recharts';

interface DataPoint {
  minute: string;
  events: number;
  errors: number;
}

export function RealTimeDashboard() {
  const [data, setData] = useState<DataPoint[]>([]);

  const fetchData = useCallback(async () => {
    const res = await fetch('/api/dashboard');
    const json: DataPoint[] = await res.json();
    setData(json.map(d => ({
      ...d,
      events: Number(d.events),
      errors: Number(d.errors),
    })));
  }, []);

  useEffect(() => {
    fetchData();
    const timer = setInterval(fetchData, 30_000);
    return () => clearInterval(timer);
  }, [fetchData]);

  return (
    <div style={{ width: '100%', height: 400 }}>
      <h2>Last 60 Minutes</h2>
      <ResponsiveContainer>
        <LineChart data={data}>
          <XAxis dataKey="minute" />
          <YAxis />
          <Tooltip />
          <Legend />
          <Line type="monotone" dataKey="events" stroke="#3b82f6" dot={false} />
          <Line type="monotone" dataKey="errors" stroke="#ef4444" dot={false} />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

## Adding a KPI Card

```tsx
function KpiCard({ label, value }: { label: string; value: number }) {
  return (
    <div className="kpi-card">
      <span className="kpi-label">{label}</span>
      <span className="kpi-value">{value.toLocaleString()}</span>
    </div>
  );
}
```

Fetch totals from a separate ClickHouse query and display alongside the chart.

## Caching on the Backend

Add a 10-second in-memory cache to prevent hammering ClickHouse when multiple dashboard tabs are open.

```typescript
let cache: { ts: number; data: unknown } | null = null;

app.get('/api/dashboard', async (req, res) => {
  if (cache && Date.now() - cache.ts < 10_000) {
    return res.json(cache.data);
  }
  const data = await queryClickHouse();
  cache = { ts: Date.now(), data };
  res.json(data);
});
```

## Summary

A ClickHouse-backed React dashboard uses a simple polling pattern: React fetches aggregated data from an API every 30 seconds, the API queries ClickHouse with time-bucketed SQL, and Recharts renders the results. Add a server-side cache to reduce ClickHouse load from concurrent dashboard viewers.
