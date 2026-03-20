# How to Deploy Typesense via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Typesense, Search, Docker, Self-Hosted

Description: Deploy Typesense fast, typo-tolerant search engine using Portainer as a developer-friendly alternative to Elasticsearch and Algolia.

## Introduction

Typesense is an open-source, typo-tolerant search engine optimized for speed and ease of use. It offers instant search, faceting, filtering, and geosearch with a simple REST API. Typesense is designed as a developer-friendly, self-hosted alternative to Algolia.

## Prerequisites

- Portainer installed with Docker

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Typesense

version: "3.8"

services:
  typesense:
    image: typesense/typesense:0.26.0.rc61
    container_name: typesense
    restart: unless-stopped
    ports:
      - "8108:8108"
    volumes:
      - typesense_data:/data
    command: >
      --data-dir /data
      --api-key=${TYPESENSE_API_KEY}
      --listen-port 8108
      --enable-cors
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8108/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - typesense_net

volumes:
  typesense_data:

networks:
  typesense_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
TYPESENSE_API_KEY=your-admin-api-key-min-16-chars
```

## Step 3: Verify the Deployment

```bash
# Check health
curl http://localhost:8108/health
# Returns: {"ok":true}

# Check server version
curl http://localhost:8108/debug \
  -H 'X-TYPESENSE-API-KEY: your-admin-api-key'
```

## Step 4: Create a Collection and Index Documents

```bash
# Create a collection (schema)
curl -X POST http://localhost:8108/collections \
  -H 'X-TYPESENSE-API-KEY: your-admin-api-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "products",
    "fields": [
      {"name": "id", "type": "string"},
      {"name": "name", "type": "string"},
      {"name": "price", "type": "float"},
      {"name": "category", "type": "string", "facet": true},
      {"name": "rating", "type": "float"}
    ],
    "default_sorting_field": "rating"
  }'

# Index documents (JSONL format)
curl -X POST "http://localhost:8108/collections/products/documents/import?action=create" \
  -H 'X-TYPESENSE-API-KEY: your-admin-api-key' \
  -H 'Content-Type: text/plain' \
  -d '{"id": "1", "name": "Laptop Pro", "price": 1299.99, "category": "Electronics", "rating": 4.8}
{"id": "2", "name": "Wireless Mouse", "price": 49.99, "category": "Accessories", "rating": 4.5}
{"id": "3", "name": "USB-C Hub", "price": 79.99, "category": "Accessories", "rating": 4.3}'
```

## Step 5: Search

```bash
# Basic search with typo tolerance
curl "http://localhost:8108/collections/products/documents/search?q=labtop&query_by=name" \
  -H 'X-TYPESENSE-API-KEY: your-admin-api-key'

# Search with facets, filter, and sort
curl "http://localhost:8108/collections/products/documents/search" \
  -H 'X-TYPESENSE-API-KEY: your-admin-api-key' \
  -G \
  --data-urlencode "q=wireless" \
  --data-urlencode "query_by=name" \
  --data-urlencode "filter_by=price:<100" \
  --data-urlencode "facet_by=category" \
  --data-urlencode "sort_by=rating:desc"
```

## Step 6: JavaScript Integration

```javascript
// npm install typesense
import Typesense from 'typesense';

const client = new Typesense.Client({
  nodes: [{ host: 'localhost', port: 8108, protocol: 'http' }],
  apiKey: 'your-admin-api-key',
  connectionTimeoutSeconds: 2,
});

// Search
const searchResults = await client.collections('products').documents().search({
  q: 'laptop',
  query_by: 'name',
  filter_by: 'price:<2000',
  sort_by: 'rating:desc',
  per_page: 10,
});
console.log(searchResults.hits);
```

## Conclusion

Typesense stores all data on disk (`/data` volume) and loads the index into memory for fast queries. The `--api-key` is the admin key - generate separate scoped API keys for client-side search to prevent users from modifying collections. Enable `--enable-cors` when Typesense is queried directly from browser JavaScript. For production, generate a read-only scoped key with field-level permissions using the `/keys` endpoint.
