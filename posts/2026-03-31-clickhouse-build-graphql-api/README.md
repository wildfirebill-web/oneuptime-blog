# How to Build a GraphQL API on Top of ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GraphQL, API Design, Apollo Server, Backend

Description: Build a GraphQL API over ClickHouse using Apollo Server and the ClickHouse Node.js client to expose analytics queries with type-safe resolvers.

---

## Why GraphQL for ClickHouse

GraphQL lets API consumers request exactly the fields they need, reducing over-fetching. For ClickHouse-backed APIs, this means resolvers only SELECT the requested columns, improving query performance and reducing network transfer. Apollo Server integrates easily with the ClickHouse client.

## Setup

```bash
npm install @apollo/server graphql @clickhouse/client
npm install --save-dev @types/node typescript
```

## Schema Definition

```javascript
// src/schema.js
const { gql } = require('graphql-tag');

const typeDefs = gql`
  type Event {
    event_time: String
    event_type: String
    user_id: String
    page_url: String
  }

  type DailyMetrics {
    date: String
    total_events: Int
    unique_users: Int
    avg_session_duration: Float
  }

  type TopPage {
    page_url: String
    view_count: Int
    unique_visitors: Int
  }

  type Query {
    events(
      start_date: String!
      end_date: String!
      event_type: String
      limit: Int
    ): [Event]

    dailyMetrics(days: Int!): [DailyMetrics]

    topPages(
      start_date: String!
      end_date: String!
      limit: Int
    ): [TopPage]
  }
`;

module.exports = typeDefs;
```

## Resolvers with ClickHouse Queries

```javascript
// src/resolvers.js
const client = require('./db');

const resolvers = {
  Query: {
    events: async (_, { start_date, end_date, event_type, limit = 50 }) => {
      const result = await client.query({
        query: `
          SELECT
            formatDateTime(event_time, '%Y-%m-%dT%H:%i:%s') AS event_time,
            event_type,
            toString(user_id) AS user_id,
            page_url
          FROM events
          WHERE event_time BETWEEN {start:DateTime} AND {end:DateTime}
            ${event_type ? 'AND event_type = {event_type:String}' : ''}
          LIMIT {limit:UInt32}
        `,
        query_params: {
          start: start_date,
          end: end_date,
          ...(event_type && { event_type }),
          limit: Math.min(limit, 1000),
        },
        format: 'JSONEachRow',
      });
      return result.json();
    },

    dailyMetrics: async (_, { days }) => {
      const result = await client.query({
        query: `
          SELECT
            toString(toDate(event_time)) AS date,
            count() AS total_events,
            toUInt32(uniq(user_id)) AS unique_users,
            avg(session_duration_sec) AS avg_session_duration
          FROM events
          WHERE event_time >= today() - {days:UInt16}
          GROUP BY date
          ORDER BY date DESC
        `,
        query_params: { days },
        format: 'JSONEachRow',
      });
      return result.json();
    },

    topPages: async (_, { start_date, end_date, limit = 10 }) => {
      const result = await client.query({
        query: `
          SELECT
            page_url,
            toUInt32(count()) AS view_count,
            toUInt32(uniq(user_id)) AS unique_visitors
          FROM events
          WHERE event_time BETWEEN {start:DateTime} AND {end:DateTime}
            AND event_type = 'page_view'
          GROUP BY page_url
          ORDER BY view_count DESC
          LIMIT {limit:UInt32}
        `,
        query_params: { start: start_date, end: end_date, limit },
        format: 'JSONEachRow',
      });
      return result.json();
    },
  },
};

module.exports = resolvers;
```

## Apollo Server Startup

```javascript
// src/server.js
const { ApolloServer } = require('@apollo/server');
const { startStandaloneServer } = require('@apollo/server/standalone');
const typeDefs = require('./schema');
const resolvers = require('./resolvers');

async function start() {
  const server = new ApolloServer({ typeDefs, resolvers });
  const { url } = await startStandaloneServer(server, {
    listen: { port: 4000 },
  });
  console.log(`GraphQL API ready at ${url}`);
}

start();
```

## Example Query

```graphql
query DashboardData {
  dailyMetrics(days: 7) {
    date
    total_events
    unique_users
  }
  topPages(start_date: "2026-03-24 00:00:00", end_date: "2026-03-31 23:59:59", limit: 5) {
    page_url
    view_count
    unique_visitors
  }
}
```

## Summary

A GraphQL API over ClickHouse uses Apollo Server resolvers that translate GraphQL queries into parameterized ClickHouse SQL. Each resolver uses `query_params` for safe parameterization, converts ClickHouse numeric types to GraphQL-compatible types, and limits result sizes. This pattern gives frontend teams a flexible, typed API over analytics data without direct database access.
