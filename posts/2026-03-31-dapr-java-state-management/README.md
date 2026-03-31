# How to Use Dapr State Management with Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Java, State Management, Spring Boot, Microservice, SDK

Description: Use the Dapr Java SDK to perform CRUD operations, bulk state, transactional state, and ETag-based concurrency control against any supported state store.

---

## Overview

Dapr state management in Java provides a consistent key-value API over Redis, Cosmos DB, PostgreSQL, and many other backends. The Java SDK returns `Mono<T>` values from Project Reactor, making it natural to use in both Spring WebFlux and traditional blocking Spring MVC applications.

## Basic CRUD Operations

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import io.dapr.client.domain.State;

public class StateExample {
    private static final String STORE = "statestore";

    public static void main(String[] args) throws Exception {
        try (DaprClient client = new DaprClientBuilder().build()) {

            // Save
            client.saveState(STORE, "order:1",
                new Order("ord-1", "Widget", 29.99)).block();

            // Get
            State<Order> state = client.getState(STORE, "order:1", Order.class).block();
            System.out.println(state.getValue().getProduct());

            // Delete
            client.deleteState(STORE, "order:1").block();
        }
    }
}
```

## ETag-Based Optimistic Concurrency

```java
// Get with ETag
State<Integer> current = client.getState(STORE, "counter", Integer.class).block();
int newValue = current.getValue() + 1;

// Try to save with ETag - fails if another update happened first
StateOptions opts = new StateOptions(
    StateOptions.Consistency.STRONG,
    StateOptions.Concurrency.FIRST_WRITE);

client.saveState(STORE, "counter", current.getEtag(),
    newValue, null, opts).block();
```

## Bulk State Operations

```java
// Bulk save
List<State<?>> states = List.of(
    new State<>("product:1", new Product("Widget", 9.99), null),
    new State<>("product:2", new Product("Gadget", 19.99), null));
client.saveBulkState(STORE, states).block();

// Bulk get
List<State<Product>> results = client.getBulkState(
    STORE,
    List.of("product:1", "product:2"),
    Product.class).block();

results.forEach(s -> System.out.printf("%s: %s%n", s.getKey(), s.getValue().getName()));
```

## Transactional State Operations

```java
import io.dapr.client.domain.TransactionalStateOperation;

List<TransactionalStateOperation<?>> ops = List.of(
    new TransactionalStateOperation<>(
        TransactionalStateOperation.OperationType.UPSERT,
        new State<>("order:2", new Order("ord-2", "Sprocket", 5.99), null)),
    new TransactionalStateOperation<>(
        TransactionalStateOperation.OperationType.DELETE,
        new State<>("cart:user-1", null, null)));

client.executeStateTransaction(STORE, ops).block();
```

## Using State in a Spring Boot Service

```java
import io.dapr.client.DaprClient;
import org.springframework.stereotype.Repository;

@Repository
public class OrderRepository {
    private static final String STORE = "statestore";
    private final DaprClient daprClient;

    public OrderRepository(DaprClient daprClient) {
        this.daprClient = daprClient;
    }

    public Order findById(String id) {
        return daprClient.getState(STORE, "order:" + id, Order.class)
            .block()
            .getValue();
    }

    public void save(Order order) {
        daprClient.saveState(STORE, "order:" + order.getId(), order).block();
    }

    public void delete(String id) {
        daprClient.deleteState(STORE, "order:" + id).block();
    }
}
```

## Summary

The Dapr Java state management API covers single key CRUD, ETag-based concurrency control, bulk operations, and ACID-like transactions. The `StateOptions` class controls consistency (eventual or strong) and concurrency (last-write-wins or first-write-wins). Because the store name is a runtime parameter, switching from Redis to Cosmos DB requires only a component YAML change.
