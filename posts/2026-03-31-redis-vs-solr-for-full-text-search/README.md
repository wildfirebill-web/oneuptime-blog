# Redis vs Solr for Full-Text Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Solr, Full-Text Search, RediSearch, Comparison, Search Engine

Description: Compare RediSearch and Apache Solr for full-text search - covering indexing, query syntax, faceting, scalability, and operational complexity.

---

Adding full-text search to an application traditionally meant deploying Solr or Elasticsearch. RediSearch (part of Redis Stack) is a newer option that runs inside Redis and handles many search requirements without a separate cluster. This post compares the two for common search workloads.

## RediSearch: Search Inside Redis

RediSearch creates inverted indexes on top of Redis hashes or JSON documents:

```bash
# Create a product search index
FT.CREATE idx:products ON HASH PREFIX 1 product: \
  SCHEMA title TEXT WEIGHT 2.0 \
         description TEXT \
         price NUMERIC SORTABLE \
         category TAG \
         in_stock TAG

# Add products
HSET product:1 title "Wireless Headphones" description "Over-ear noise cancelling" \
     price 149.99 category "electronics" in_stock "true"
HSET product:2 title "Wired Headphones" description "Studio quality sound" \
     price 49.99 category "electronics" in_stock "true"

# Search
FT.SEARCH idx:products "headphones" FILTER price 0 100

# Aggregation
FT.AGGREGATE idx:products "*" GROUPBY 1 @category REDUCE COUNT 0 AS count
```

In Python:

```python
from redis.commands.search.field import TextField, NumericField, TagField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = [
    TextField("title", weight=2.0),
    TextField("description"),
    NumericField("price", sortable=True),
    TagField("category"),
]

client.ft("idx:products").create_index(
    schema,
    definition=IndexDefinition(prefix=["product:"], index_type=IndexType.HASH),
)

results = client.ft("idx:products").search("headphones")
for doc in results.docs:
    print(doc.title, doc.price)
```

## Apache Solr: Mature Full-Text Search

Solr is a standalone search server built on Lucene. Configuration is schema-based:

```xml
<!-- schema.xml snippet -->
<field name="title" type="text_en" indexed="true" stored="true" />
<field name="price" type="pfloat" indexed="true" stored="true" sortMissingLast="true" />
<field name="category" type="string" indexed="true" stored="true" multiValued="true" />
```

Indexing via HTTP:

```bash
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '[{"id":"1","title":"Wireless Headphones","price":149.99,"category":"electronics"}]'
```

Query with faceting:

```bash
curl "http://localhost:8983/solr/products/select?q=headphones&fq=price:[0+TO+100]&facet=true&facet.field=category"
```

Python with `pysolr`:

```python
import pysolr

solr = pysolr.Solr("http://localhost:8983/solr/products", timeout=10)
results = solr.search("headphones", fq="price:[0 TO 100]", rows=10)
for result in results:
    print(result["title"])
```

## Comparison Table

| Feature | RediSearch | Apache Solr |
|---------|-----------|-------------|
| Deployment | Redis module (embedded) | Standalone server |
| Indexing latency | < 1 ms | 10-100 ms |
| Query latency | < 1 ms | 5-50 ms |
| Full-text search | Yes | Yes (Lucene) |
| Fuzzy search | Yes | Yes |
| Faceting | Yes | Yes (mature) |
| Geospatial | Yes (GEODIST) | Yes |
| Spell checking | Basic | Advanced |
| Join queries | No | Yes (join parser) |
| Horizontal scaling | Redis Cluster | SolrCloud + ZooKeeper |
| Ops complexity | Low | High |

## When to Use RediSearch

- You already run Redis and want to avoid a separate search cluster.
- Your dataset fits in RAM (under 50 GB of indexed data).
- You need sub-millisecond query latency for autocomplete or type-ahead.
- Simple full-text search with filters and facets.

## When to Use Solr

- You need advanced Lucene features: complex tokenizers, synonym files, spell check.
- Your corpus is large (hundreds of GB) and disk-based indexing is required.
- You need join-style queries across collections.
- Your team has existing Solr expertise.

## Summary

RediSearch is the practical choice when you want search capability without deploying and operating a separate search cluster, especially for sub-millisecond latency needs. Solr's maturity and rich Lucene feature set make it the better choice for large document corpora, complex linguistic processing, or when your team already manages SolrCloud. For most product catalogs and content search under 10 million documents, RediSearch performs well and eliminates operational overhead.
