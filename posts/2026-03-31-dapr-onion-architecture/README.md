# How to Use Dapr with Onion Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Onion Architecture, Design Pattern, Dependency Inversion, Microservice

Description: Structure Dapr microservices using Onion Architecture with domain at the center and Dapr building blocks in the outer infrastructure layer behind interface abstractions.

---

## Onion Architecture Overview

Onion Architecture organizes code in concentric rings where dependencies point inward:
- **Domain Model** (center) - entities, value objects, domain events
- **Domain Services** - business logic using domain interfaces
- **Application Services** - orchestrates use cases
- **Infrastructure** (outer) - Dapr implementations, HTTP controllers

Dapr building blocks live in the outermost ring, injected inward through interfaces.

## Layer Structure

```text
src/
- Domain/                         # Center - no external dependencies
  - Models/Order.java
  - Events/OrderCreatedEvent.java
  - Repositories/IOrderRepository.java
  - Services/OrderDomainService.java
- Application/
  - Services/OrderApplicationService.java
  - Ports/IOrderEventPort.java
- Infrastructure/                 # Outer ring - Dapr implementations
  - Dapr/
    - DaprOrderRepository.java
    - DaprOrderEventAdapter.java
  - Web/
    - OrderController.java
```

## Domain Layer - Pure Java/Kotlin, No Dapr

```java
// Domain/Repositories/IOrderRepository.java
public interface IOrderRepository {
    Optional<Order> findById(String orderId);
    void save(Order order);
    void delete(String orderId);
}

// Domain/Models/Order.java
public class Order {
    private final String id;
    private final String customerId;
    private OrderStatus status;
    private final List<String> items;

    public Order(String customerId, List<String> items) {
        this.id = UUID.randomUUID().toString();
        this.customerId = customerId;
        this.status = OrderStatus.CREATED;
        this.items = List.copyOf(items);
    }

    public void confirm() {
        if (status != OrderStatus.CREATED) {
            throw new IllegalStateException("Cannot confirm order in status: " + status);
        }
        this.status = OrderStatus.CONFIRMED;
    }

    // Getters...
}
```

## Application Service Layer

```java
// Application/Services/OrderApplicationService.java
@Service
public class OrderApplicationService {
    private final IOrderRepository orderRepository;
    private final IOrderEventPort eventPort;

    public OrderApplicationService(
            IOrderRepository orderRepository,
            IOrderEventPort eventPort) {
        this.orderRepository = orderRepository;
        this.eventPort = eventPort;
    }

    public String createOrder(String customerId, List<String> items) {
        Order order = new Order(customerId, items);
        orderRepository.save(order);
        eventPort.publishOrderCreated(order.getId(), customerId);
        return order.getId();
    }
}
```

## Infrastructure Layer - Dapr Implementations

```java
// Infrastructure/Dapr/DaprOrderRepository.java
@Component
public class DaprOrderRepository implements IOrderRepository {
    private final DaprClient daprClient;
    private static final String STORE_NAME = "statestore";

    public DaprOrderRepository(DaprClient daprClient) {
        this.daprClient = daprClient;
    }

    @Override
    public Optional<Order> findById(String orderId) {
        Order order = daprClient.getState(STORE_NAME, orderId, Order.class).block().getValue();
        return Optional.ofNullable(order);
    }

    @Override
    public void save(Order order) {
        daprClient.saveState(STORE_NAME, order.getId(), order).block();
    }

    @Override
    public void delete(String orderId) {
        daprClient.deleteState(STORE_NAME, orderId).block();
    }
}

// Infrastructure/Dapr/DaprOrderEventAdapter.java
@Component
public class DaprOrderEventAdapter implements IOrderEventPort {
    private final DaprClient daprClient;

    public DaprOrderEventAdapter(DaprClient daprClient) {
        this.daprClient = daprClient;
    }

    @Override
    public void publishOrderCreated(String orderId, String customerId) {
        Map<String, String> event = Map.of(
            "orderId", orderId,
            "customerId", customerId
        );
        daprClient.publishEvent("pubsub", "order-created", event).block();
    }
}
```

## Spring Boot Configuration

```java
// Infrastructure/Config/DaprConfig.java
@Configuration
public class DaprConfig {

    @Bean
    public DaprClient daprClient() {
        return new DaprClientBuilder().build();
    }
}
```

## Unit Test Without Dapr

```java
@Test
void createOrder_ShouldSaveAndPublish() {
    IOrderRepository mockRepo = mock(IOrderRepository.class);
    IOrderEventPort mockPort = mock(IOrderEventPort.class);

    OrderApplicationService service = new OrderApplicationService(mockRepo, mockPort);
    String orderId = service.createOrder("cust-1", List.of("item-a"));

    verify(mockRepo).save(argThat(o -> o.getCustomerId().equals("cust-1")));
    verify(mockPort).publishOrderCreated(orderId, "cust-1");
}
```

## Summary

Onion Architecture applied to Dapr microservices ensures the domain model stays free of infrastructure concerns by placing Dapr implementations in the outermost layer behind interfaces. Dependencies flow inward through the application service into the domain, making business logic fully testable without a running Dapr sidecar and enabling component swaps without domain changes.
