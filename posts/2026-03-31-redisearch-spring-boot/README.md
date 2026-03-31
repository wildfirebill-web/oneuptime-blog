# How to Use RediSearch with Spring Boot

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Spring Boot, RediSearch, Search, Java

Description: Add full-text search and secondary indexing to Spring Boot applications using RediSearch via redis-om-spring or Jedis search commands.

---

RediSearch adds full-text search, numeric filters, and geo queries on top of Redis data structures. Combined with Spring Boot, it replaces heavyweight search infrastructure for many use cases.

## Option 1: redis-om-spring (Recommended)

### Add Dependency

```xml
<dependency>
    <groupId>com.redis.om</groupId>
    <artifactId>redis-om-spring</artifactId>
    <version>0.9.0</version>
</dependency>
```

### Define a Searchable Entity

```java
import com.redis.om.spring.annotations.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;

@RedisHash("Product")
public class Product {

    @Id
    private String id;

    @Searchable
    private String name;

    @Searchable
    private String description;

    @Indexed
    private String category;

    @Indexed
    private double price;

    // getters and setters
}
```

`@Searchable` enables full-text search. `@Indexed` enables exact match and numeric range queries.

### Create the Repository

```java
import com.redis.om.spring.repository.RedisEnhancedRepository;

public interface ProductRepository extends RedisEnhancedRepository<Product, String> {
    // Full-text search
    SearchResult<Product> search(String query);
    // Derived queries
    List<Product> findByCategory(String category);
    List<Product> findByPriceBetween(double min, double max);
}
```

### Enable and Use

```java
@SpringBootApplication
@EnableRedisEnhancedRepositories(basePackages = "com.example")
public class Application {}
```

```java
@RestController
public class SearchController {

    private final ProductRepository repo;

    @GetMapping("/search")
    public List<Product> search(@RequestParam String q,
                                @RequestParam String category) {
        return repo.findByCategory(category);
    }
}
```

## Option 2: Jedis Raw Search Commands

```java
@Bean
public UnifiedJedis jedis() {
    return new JedisPooled("localhost", 6379);
}

@Service
public class SearchService {

    private final UnifiedJedis jedis;

    public void createIndex() {
        Schema schema = new Schema()
            .addTextField("name", 5.0)
            .addTextField("description", 1.0)
            .addTagField("category")
            .addNumericField("price");

        IndexDefinition def = new IndexDefinition()
            .setPrefixes("Product:");
        jedis.ftCreate("product-idx", IndexOptions.defaultOptions().setDefinition(def), schema);
    }

    public SearchResult search(String query) {
        return jedis.ftSearch("product-idx", new Query(query).limit(0, 20));
    }

    public SearchResult filter(String category, double minPrice, double maxPrice) {
        Query q = new Query("@category:{" + category + "} @price:[" + minPrice + " " + maxPrice + "]");
        return jedis.ftSearch("product-idx", q);
    }
}
```

## Inspect Index in Redis CLI

```bash
redis-cli ft.info product-idx
redis-cli ft.search product-idx "@category:{electronics} @price:[100 500]" LIMIT 0 10
```

## Summary

RediSearch turns Redis into a capable search engine for Spring Boot applications. redis-om-spring abstracts the index creation and query building behind familiar annotations and derived query methods. For cases needing raw control, Jedis exposes the full RediSearch command set. Both approaches deliver sub-millisecond query latency without deploying a separate Elasticsearch cluster.
