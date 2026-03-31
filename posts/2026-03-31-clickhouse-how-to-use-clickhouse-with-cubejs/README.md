# How to Use ClickHouse with Cube.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cube.js, Analytics, Data Modeling, OLAP, Business Intelligence

Description: Learn how to connect Cube.js to ClickHouse to build a semantic data layer and serve fast, cached analytical queries to your frontend applications.

---

Cube.js is a headless business intelligence framework that sits between your database and your frontend. When paired with ClickHouse, it provides a powerful semantic layer with built-in caching, pre-aggregations, and a REST or GraphQL API for analytics.

## Installing Cube.js and the ClickHouse Driver

Start a new Cube.js project and install the ClickHouse database driver.

```bash
npx cubejs-cli create my-cube-app -d clickhouse
cd my-cube-app
npm install
```

## Configuring the ClickHouse Connection

Edit `cube.js` (or `.env`) to point to your ClickHouse instance.

```bash
CUBEJS_DB_TYPE=clickhouse
CUBEJS_DB_HOST=localhost
CUBEJS_DB_PORT=8123
CUBEJS_DB_NAME=default
CUBEJS_DB_USER=default
CUBEJS_DB_PASS=
```

## Defining a Cube Schema

Create a schema file `schema/PageViews.js` to model a ClickHouse table.

```javascript
cube(`PageViews`, {
  sql: `SELECT * FROM default.page_views`,

  measures: {
    count: {
      type: `count`,
    },
    uniqueUsers: {
      sql: `user_id`,
      type: `countDistinct`,
    },
  },

  dimensions: {
    page: {
      sql: `page`,
      type: `string`,
    },
    createdAt: {
      sql: `created_at`,
      type: `time`,
    },
  },
});
```

## Enabling Pre-Aggregations

Pre-aggregations allow Cube.js to materialize query results into rollup tables for sub-second response times.

```javascript
preAggregations: {
  main: {
    type: `rollup`,
    measures: [PageViews.count, PageViews.uniqueUsers],
    dimensions: [PageViews.page],
    timeDimension: PageViews.createdAt,
    granularity: `day`,
  },
},
```

ClickHouse's columnar storage and pre-aggregations together make analytics extremely fast.

## Querying via the REST API

Once Cube.js is running, send queries using its JSON API.

```bash
curl -X POST http://localhost:4000/cubejs-api/v1/load \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "measures": ["PageViews.count"],
      "dimensions": ["PageViews.page"],
      "timeDimensions": [{
        "dimension": "PageViews.createdAt",
        "granularity": "day"
      }]
    }
  }'
```

## Connecting a Frontend

Cube.js integrates with React, Vue, and plain JavaScript via its client libraries.

```bash
npm install @cubejs-client/core @cubejs-client/react
```

```javascript
import cubejs from '@cubejs-client/core';

const cubejsApi = cubejs('YOUR_TOKEN', {
  apiUrl: 'http://localhost:4000/cubejs-api/v1',
});

cubejsApi.load({
  measures: ['PageViews.count'],
  dimensions: ['PageViews.page'],
}).then(resultSet => {
  console.log(resultSet.tablePivot());
});
```

## Summary

Cube.js combined with ClickHouse gives you a production-ready analytics stack. ClickHouse handles fast columnar storage and aggregation, while Cube.js provides schema modeling, caching, pre-aggregations, and a developer-friendly API layer for building dashboards and reports.
