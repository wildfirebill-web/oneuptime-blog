# How to Use RediSearch with Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, RediSearch

Description: Learn how to create RediSearch indexes, index documents, and run full-text and numeric queries in Java using Jedis.

---

RediSearch adds full-text search, secondary indexing, and aggregations to Redis. Jedis ships with a built-in API for RediSearch commands, making it straightforward to index and query structured data stored as Redis hashes or JSON documents.

## Add Jedis Dependency

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.1.0</version>
</dependency>
```

## Create an Index

```java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Schema;
import redis.clients.jedis.search.IndexDefinition;
import redis.clients.jedis.search.IndexOptions;

UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379");

Schema schema = new Schema()
    .addTextField("title", 5.0)     // weight 5 for title
    .addTextField("body", 1.0)
    .addNumericField("price")
    .addTagField("category");

IndexDefinition def = new IndexDefinition()
    .setPrefixes("product:");

jedis.ftCreate("product-index",
    IndexOptions.defaultOptions().setDefinition(def),
    schema);
```

## Index Documents

RediSearch indexes Redis hashes automatically based on key prefixes. Add documents as hashes:

```java
Map<String, String> product1 = new HashMap<>();
product1.put("title", "Wireless Headphones");
product1.put("body", "Noise-cancelling over-ear headphones");
product1.put("price", "99.99");
product1.put("category", "electronics");
jedis.hset("product:1", product1);

Map<String, String> product2 = new HashMap<>();
product2.put("title", "Running Shoes");
product2.put("body", "Lightweight shoes for trail running");
product2.put("price", "79.99");
product2.put("category", "sports");
jedis.hset("product:2", product2);
```

## Full-Text Search

```java
import redis.clients.jedis.search.Query;
import redis.clients.jedis.search.SearchResult;

// Basic full-text query
SearchResult result = jedis.ftSearch("product-index", new Query("headphones"));
result.getDocuments().forEach(doc -> {
    System.out.println(doc.getId() + ": " + doc.get("title"));
});
```

## Numeric and Tag Filters

```java
// Price range filter
Query priceQuery = new Query("@price:[50 100]");
SearchResult affordable = jedis.ftSearch("product-index", priceQuery);

// Tag filter
Query categoryQuery = new Query("@category:{electronics}");
SearchResult electronics = jedis.ftSearch("product-index", categoryQuery);

// Combined query
Query combined = new Query("running @price:[0 100]");
SearchResult results = jedis.ftSearch("product-index", combined);
```

## Pagination and Sorting

```java
Query pagedQuery = new Query("*")
    .limit(0, 10)           // offset 0, return 10 results
    .setSortBy("price", true); // ascending

SearchResult page1 = jedis.ftSearch("product-index", pagedQuery);
System.out.println("Total: " + page1.getTotalResults());
```

## Aggregations

```java
import redis.clients.jedis.search.aggr.*;

AggregationBuilder agg = new AggregationBuilder("*")
    .groupBy("@category", Reducers.count().as("count"))
    .sortBy(new SortedField("@count", SortedField.SortOrder.DESC));

AggregationResult aggResult = jedis.ftAggregate("product-index", agg);
aggResult.getRows().forEach(row -> {
    System.out.println(row.getString("category") + ": " + row.getLong("count"));
});
```

## Summary

RediSearch with Jedis lets you build full-text search and analytics on top of existing Redis hash data. Creating an index with a prefix filter means documents are indexed automatically as hashes are written. Queries support free-text, numeric ranges, tag filters, pagination, sorting, and aggregations - all executing within Redis without an external search engine.
