# How to Deploy Meilisearch via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Meilisearch, Search, Docker, Self-Hosted

Description: Deploy Meilisearch fast, open-source search engine using Portainer for full-text search with typo tolerance.

## Introduction

Meilisearch is a fast, open-source, RESTful search engine with typo tolerance, faceting, filtering, and geosearch built in. It is designed for end-user search experiences and requires no query language to operate.

## Prerequisites

- Portainer installed with Docker

## Step 1: Create the Stack in Portainer

Navigate to **Stacks** > **Add Stack**:

```yaml
# docker-compose.yml - Meilisearch

version: "3.8"

services:
  meilisearch:
    image: getmeili/meilisearch:v1.8.3
    container_name: meilisearch
    restart: unless-stopped
    ports:
      - "7700:7700"
    volumes:
      - meilisearch_data:/meili_data
    environment:
      - MEILI_MASTER_KEY=${MEILI_MASTER_KEY}
      - MEILI_ENV=production
      - MEILI_DB_PATH=/meili_data
      - MEILI_HTTP_ADDR=0.0.0.0:7700
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:7700/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - meilisearch_net

volumes:
  meilisearch_data:

networks:
  meilisearch_net:
    driver: bridge
```

## Step 2: Set Environment Variables in Portainer

```text
MEILI_MASTER_KEY=your-master-key-min-16-chars
```

When `MEILI_ENV=production`, the master key is required and all API endpoints require authentication.

## Step 3: Verify the Deployment

```bash
# Check health (no auth needed for /health)
curl http://localhost:7700/health
# Returns: {"status":"available"}

# Get server version
curl http://localhost:7700/version \
  -H 'Authorization: Bearer your-master-key'
```

## Step 4: Index Documents

```bash
# Create an index and add documents
curl -X POST http://localhost:7700/indexes/movies/documents \
  -H 'Authorization: Bearer your-master-key' \
  -H 'Content-Type: application/json' \
  -d '[
    {"id": 1, "title": "Carol", "genre": "Drama", "year": 2015},
    {"id": 2, "title": "Wonder Woman", "genre": "Action", "year": 2017},
    {"id": 3, "title": "Life of Pi", "genre": "Drama", "year": 2012}
  ]'

# Check indexing task status
curl http://localhost:7700/tasks/0 \
  -H 'Authorization: Bearer your-master-key'
```

## Step 5: Search

```bash
# Basic search with typo tolerance
curl 'http://localhost:7700/indexes/movies/search' \
  -H 'Authorization: Bearer your-master-key' \
  -H 'Content-Type: application/json' \
  -d '{"q": "caroll"}'

# Search with filter and facets
curl 'http://localhost:7700/indexes/movies/search' \
  -H 'Authorization: Bearer your-master-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "q": "drama",
    "filter": "year > 2013",
    "sort": ["year:desc"],
    "limit": 5
  }'
```

## Step 6: Python Integration

```python
# pip install meilisearch
import meilisearch

client = meilisearch.Client('http://localhost:7700', 'your-master-key')

# Index documents
index = client.index('products')
documents = [
    {"id": 1, "name": "Laptop Pro", "brand": "TechCorp", "price": 1299},
    {"id": 2, "name": "Wireless Mouse", "brand": "InputCo", "price": 49},
]
task = index.add_documents(documents)
client.wait_for_task(task.task_uid)

# Search
results = index.search("laptop", {
    "filter": "price < 1500",
    "sort": ["price:asc"],
    "limit": 10,
})
print(results["hits"])
```

## Conclusion

Meilisearch stores all data and indexes in the `MEILI_DB_PATH` directory. In `production` mode, always set a `MEILI_MASTER_KEY` - all requests must include an `Authorization: Bearer <key>` header. Use the Meilisearch dashboard (available in development mode at `http://localhost:7700`) to explore indexes and test queries. Generate scoped API keys for client-side search to prevent users from accessing admin endpoints.
