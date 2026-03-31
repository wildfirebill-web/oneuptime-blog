# How to Use Spring Cache Abstraction with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Spring Boot, Cache, Java, Performance

Description: Cache Spring Boot service methods with Redis using @Cacheable, @CachePut, and @CacheEvict annotations with TTL and serialisation configuration.

---

Spring's cache abstraction lets you cache method return values with a single annotation. Backing it with Redis makes the cache distributed and shared across all application instances.

## Add Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

## Enable Caching and Configure Redis

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofMinutes(5)))
            .build();
    }
}
```

## Use Cache Annotations

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        // Called only on cache miss
        return productRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    @CacheEvict(value = "products", allEntries = true)
    public void clearAll() {
        // Clears all entries in the cache
    }
}
```

`@Cacheable` checks the cache first. `@CachePut` always runs the method and updates the cache. `@CacheEvict` removes entries on mutation.

## Conditional Caching

Cache only when results meet a condition:

```java
@Cacheable(value = "products", key = "#id", condition = "#id > 0", unless = "#result == null")
public Product findById(Long id) {
    return productRepository.findById(id).orElse(null);
}
```

## Verify Cache Entries

```bash
redis-cli keys "products::*"
redis-cli ttl "products::42"
redis-cli get "products::42"
```

## application.yml

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
```

## Summary

Spring's cache abstraction with Redis requires almost no boilerplate - annotations handle cache population, update, and eviction. Configuring `RedisCacheManager` with per-cache TTLs and JSON serialisation ensures entries expire correctly and remain readable across deployments. Distributing the cache via Redis means all instances share the same warm cache.
