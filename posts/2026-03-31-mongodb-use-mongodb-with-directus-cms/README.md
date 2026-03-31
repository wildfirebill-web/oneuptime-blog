# How to Use MongoDB with Directus CMS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Directus, CMS

Description: Set up Directus headless CMS with MongoDB as the data store, configure collections, and use the Directus SDK to query your content programmatically.

---

## Directus and MongoDB

Directus is a headless CMS and data platform that wraps any SQL or NoSQL database with a REST and GraphQL API. With MongoDB as the backend, Directus provides a visual data management interface over your document collections, auto-generates CRUD endpoints, and handles authentication - all without writing boilerplate.

Note: Directus requires a driver-level MongoDB integration. As of Directus v10+, MongoDB support is provided through the `knex-mongodb` driver or a dedicated adapter. Confirm compatibility with your Directus version before deploying.

## Installation

```bash
npm init directus-project@latest my-cms
# Select MongoDB when prompted for database type
```

Or configure manually in `.env`:

```text
DB_CLIENT=mongodb
DB_CONNECTION_STRING=mongodb://localhost:27017/directus
DB_DATABASE=directus
```

## Docker Compose Setup

```yaml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: directus

  directus:
    image: directus/directus:latest
    ports:
      - "8055:8055"
    depends_on:
      - mongodb
    environment:
      DB_CLIENT: mongodb
      DB_CONNECTION_STRING: "mongodb://mongodb:27017/directus"
      SECRET: "your-secret-key"
      ADMIN_EMAIL: admin@example.com
      ADMIN_PASSWORD: adminpassword
      PUBLIC_URL: http://localhost:8055

volumes:
  mongo_data:
```

Start with:

```bash
docker-compose up -d
```

## Creating a Collection via API

```bash
curl -X POST http://localhost:8055/collections \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "collection": "articles",
    "fields": [
      { "field": "title", "type": "string" },
      { "field": "content", "type": "text" },
      { "field": "published_at", "type": "dateTime" }
    ]
  }'
```

## Querying Content with the Directus SDK

```javascript
import { createDirectus, rest, readItems } from '@directus/sdk'

const client = createDirectus('http://localhost:8055').with(rest())

// Fetch published articles sorted by date
const articles = await client.request(
  readItems('articles', {
    filter: {
      published_at: { _nnull: true },
    },
    sort: ['-published_at'],
    limit: 20,
    fields: ['id', 'title', 'published_at'],
  })
)

console.log(articles)
```

## Custom MongoDB Aggregation via Directus Hooks

For complex queries, use Directus hooks to run native MongoDB aggregations:

```javascript
// extensions/hooks/custom-stats/index.js
export default ({ filter, action }, { database }) => {
  action('articles.read', async ({ collection }, { schema }) => {
    const db = database.client.config.connection
    const stats = await db.collection('articles').aggregate([
      { $group: { _id: null, total: { $sum: 1 } } },
    ]).toArray()
    console.log('Total articles:', stats[0]?.total)
  })
}
```

## Configuring Roles and Permissions

```bash
curl -X POST http://localhost:8055/permissions \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "role": "PUBLIC_ROLE_ID",
    "collection": "articles",
    "action": "read",
    "fields": ["title", "content", "published_at"],
    "filter": { "published_at": { "_nnull": true } }
  }'
```

## Summary

Directus with MongoDB gives you a full-featured headless CMS without writing custom backend code. Configure the connection through environment variables, create collections via the API or admin UI, and query content using the Directus SDK. For complex MongoDB-native queries, Directus hooks expose the underlying database connection so you can run aggregations alongside Directus's standard CRUD operations.
