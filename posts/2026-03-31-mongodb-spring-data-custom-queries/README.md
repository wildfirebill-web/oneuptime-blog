# How to Use Spring Data MongoDB Custom Queries with @Query

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Spring, Query, Spring Data, Annotation

Description: Learn how to write custom MongoDB queries in Spring Data using the @Query annotation with JSON filter expressions and field projections.

---

## When to Use @Query

Spring Data MongoDB's derived query method names handle simple cases well, but complex queries with nested conditions, array operators, or projections quickly become unwieldy as method names. The `@Query` annotation lets you write raw MongoDB query documents directly in your repository interface.

## Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/shopdb
```

## Basic @Query Usage

Parameters are referenced as `?0`, `?1`, etc. (zero-indexed):

```java
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import java.util.List;

public interface OrderRepository extends MongoRepository<Order, String> {

    @Query("{ 'status': ?0, 'total': { $gte: ?1 } }")
    List<Order> findByStatusAndMinTotal(String status, double minTotal);

    @Query("{ 'customerId': ?0, 'createdAt': { $gte: ?1, $lte: ?2 } }")
    List<Order> findByCustomerBetweenDates(
        String customerId, Date from, Date to);
}
```

## Field Projections

Use the `fields` attribute to return only specific fields:

```java
// Return only name and price fields
@Query(value = "{ 'category': ?0 }",
       fields = "{ 'name': 1, 'price': 1 }")
List<Product> findProductSummaryByCategory(String category);

// Exclude a specific field
@Query(value = "{}",
       fields = "{ 'internalNotes': 0 }")
List<Product> findAllPublicProducts();
```

## Array Queries

```java
// Match documents where the tags array contains a value
@Query("{ 'tags': { $in: ?0 } }")
List<Article> findByTagsIn(List<String> tags);

// Match documents where all provided tags are present
@Query("{ 'tags': { $all: ?0 } }")
List<Article> findByAllTags(List<String> tags);

// Array element matching
@Query("{ 'lineItems': { $elemMatch: { 'productId': ?0, 'quantity': { $gte: ?1 } } } }")
List<Order> findOrdersWithProductQuantity(String productId, int minQty);
```

## Using SpEL Expressions

Spring Data MongoDB supports Spring Expression Language (SpEL) in `@Query` via `?#`:

```java
@Query("{ 'category': ?#{[0].toLowerCase()}, 'price': { $lte: ?1 } }")
List<Product> findByCategoryIgnoreCase(String category, double maxPrice);
```

## Sorting and Pagination

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;

@Query("{ 'status': ?0 }")
List<Order> findByStatus(String status, Sort sort);

@Query("{ 'status': ?0 }")
Page<Order> findByStatusPaged(String status, Pageable pageable);
```

Calling code:

```java
Page<Order> pending = orderRepo.findByStatusPaged("PENDING",
    PageRequest.of(0, 20, Sort.by("createdAt").descending()));
```

## Count and Exists Queries

```java
@Query(value = "{ 'customerId': ?0, 'status': 'OPEN' }", count = true)
long countOpenOrders(String customerId);

@Query(value = "{ 'sku': ?0 }", exists = true)
boolean productExists(String sku);
```

## Delete Queries

```java
@Query(value = "{ 'status': 'CANCELLED', 'createdAt': { $lt: ?0 } }",
       delete = true)
long deleteOldCancelledOrders(Date cutoff);
```

## Combining with @Aggregation

For pipeline-style transformations, use `@Aggregation` alongside `@Query`:

```java
import org.springframework.data.mongodb.repository.Aggregation;

@Aggregation(pipeline = {
    "{ $match: { 'status': ?0 } }",
    "{ $group: { _id: '$customerId', total: { $sum: '$amount' } } }",
    "{ $sort: { total: -1 } }",
    "{ $limit: 10 }"
})
List<CustomerTotal> topCustomersByStatus(String status);
```

## Summary

The `@Query` annotation in Spring Data MongoDB lets you embed MongoDB query documents directly in repository interfaces, supporting filter expressions, field projections, array operators, SpEL expressions, and pagination. Use `count = true`, `exists = true`, or `delete = true` attributes to change the operation type. For multi-stage transformations, the `@Aggregation` annotation provides full pipeline support without leaving the repository abstraction.
