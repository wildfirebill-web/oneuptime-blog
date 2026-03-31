# How to Use MySQL with Hibernate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Hibernate, Java, ORM, JPA

Description: Learn how to configure Hibernate with MySQL in a Java application, map entities, manage sessions, and perform CRUD operations using HQL and the Criteria API.

---

## Introduction

Hibernate is the most widely used ORM framework for Java. It implements the JPA specification and maps Java entity classes to MySQL tables, handling SQL generation, lazy loading, caching, and transaction management. This guide covers standalone Hibernate setup with MySQL.

## Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>6.4.0.Final</version>
    </dependency>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.2.0</version>
    </dependency>
</dependencies>
```

## Hibernate Configuration (hibernate.cfg.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/mydb?useSSL=false&amp;serverTimezone=UTC</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">password</property>
        <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
        <property name="hibernate.hbm2ddl.auto">update</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.connection.pool_size">10</property>
        <mapping class="com.example.Product"/>
    </session-factory>
</hibernate-configuration>
```

## Defining an Entity

```java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_category_price", columnList = "category_id, price")
})
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    // getters and setters omitted for brevity
}
```

## CRUD Operations with SessionFactory

```java
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

SessionFactory sessionFactory = new Configuration()
    .configure("hibernate.cfg.xml")
    .buildSessionFactory();

// Create
try (Session session = sessionFactory.openSession()) {
    session.beginTransaction();
    Product product = new Product();
    product.setName("Laptop");
    product.setPrice(new BigDecimal("999.99"));
    product.setStock(50);
    session.persist(product);
    session.getTransaction().commit();
}

// Read with HQL
try (Session session = sessionFactory.openSession()) {
    List<Product> products = session.createQuery(
        "FROM Product p WHERE p.price < :maxPrice ORDER BY p.price",
        Product.class
    ).setParameter("maxPrice", new BigDecimal("1000")).getResultList();
    products.forEach(p -> System.out.println(p.getName()));
}

// Update
try (Session session = sessionFactory.openSession()) {
    session.beginTransaction();
    session.createMutationQuery(
        "UPDATE Product SET stock = stock + 10 WHERE stock < 5"
    ).executeUpdate();
    session.getTransaction().commit();
}
```

## Using the JPA Criteria API

```java
try (Session session = sessionFactory.openSession()) {
    CriteriaBuilder cb = session.getCriteriaBuilder();
    CriteriaQuery<Product> cq = cb.createQuery(Product.class);
    Root<Product> root = cq.from(Product.class);

    cq.select(root)
      .where(cb.and(
          cb.greaterThan(root.get("stock"), 0),
          cb.lessThan(root.get("price"), new BigDecimal("500"))
      ))
      .orderBy(cb.asc(root.get("price")));

    List<Product> results = session.createQuery(cq).getResultList();
}
```

## Summary

Hibernate maps Java entities to MySQL tables through annotations and manages sessions via `SessionFactory`. Use HQL for readable queries, the Criteria API for type-safe dynamic queries, and `@PrePersist` hooks for lifecycle events. Set `hibernate.hbm2ddl.auto=validate` in production to prevent accidental schema changes.
