# How to Build Dapr Workflows with Java SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflows, Java, Spring Boot, Orchestration

Description: Learn how to build durable, fault-tolerant workflow orchestrations using the Dapr Workflow API with the Java SDK and Spring Boot.

---

## Overview of Dapr Workflows in Java

Dapr Workflows with the Java SDK allow you to write durable orchestrations as regular Java classes. The Dapr runtime persists the execution state and can replay from checkpoints after any failure. This makes it suitable for long-running business processes like order fulfillment, approval chains, and data pipelines.

## Project Setup with Maven

Add the Dapr SDK dependency to your `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>io.dapr</groupId>
        <artifactId>dapr-sdk-springboot</artifactId>
        <version>1.11.0</version>
    </dependency>
    <dependency>
        <groupId>io.dapr</groupId>
        <artifactId>dapr-sdk-workflows</artifactId>
        <version>1.11.0</version>
    </dependency>
</dependencies>
```

## Defining Workflow Activities

Activities are Java classes extending `WorkflowActivity`:

```java
import io.dapr.workflows.runtime.WorkflowActivity;
import io.dapr.workflows.runtime.WorkflowActivityContext;

public class ReserveInventoryActivity extends WorkflowActivity {

    @Override
    public Object run(WorkflowActivityContext ctx) {
        InventoryRequest request = ctx.getInput(InventoryRequest.class);
        System.out.printf("Reserving %d units of %s%n", request.getQuantity(), request.getSku());

        // Business logic here
        boolean reserved = inventoryService.reserve(request.getSku(), request.getQuantity());

        return new InventoryResult(reserved, request.getSku());
    }
}
```

```java
public class ChargeCustomerActivity extends WorkflowActivity {

    @Override
    public Object run(WorkflowActivityContext ctx) {
        PaymentRequest request = ctx.getInput(PaymentRequest.class);
        System.out.printf("Charging customer %s for $%.2f%n",
            request.getCustomerId(), request.getAmount());

        String transactionId = paymentGateway.charge(
            request.getCustomerId(), request.getAmount());

        return new PaymentResult(transactionId != null, transactionId);
    }
}
```

## Defining a Workflow

Workflows are Java classes extending `Workflow`. The `run` method defines the orchestration:

```java
import io.dapr.workflows.Workflow;
import io.dapr.workflows.WorkflowStub;

public class OrderFulfillmentWorkflow extends Workflow {

    @Override
    public WorkflowStub create() {
        return ctx -> {
            ctx.getLogger().info("Starting order fulfillment workflow");

            OrderRequest order = ctx.getInput(OrderRequest.class);

            // Step 1: Reserve inventory
            InventoryResult inventoryResult = ctx.callActivity(
                ReserveInventoryActivity.class.getName(),
                new InventoryRequest(order.getSku(), order.getQuantity()),
                InventoryResult.class
            ).await();

            if (!inventoryResult.isSuccess()) {
                ctx.complete(new OrderResult(false, "Inventory unavailable"));
                return;
            }

            // Step 2: Charge the customer
            PaymentResult paymentResult = ctx.callActivity(
                ChargeCustomerActivity.class.getName(),
                new PaymentRequest(order.getCustomerId(), order.getTotal()),
                PaymentResult.class
            ).await();

            if (!paymentResult.isSuccess()) {
                // Compensate - release inventory
                ctx.callActivity(
                    ReleaseInventoryActivity.class.getName(),
                    new InventoryRequest(order.getSku(), order.getQuantity()),
                    Void.class
                ).await();
                ctx.complete(new OrderResult(false, "Payment failed"));
                return;
            }

            // Step 3: Notify customer
            ctx.callActivity(
                NotifyCustomerActivity.class.getName(),
                new NotificationRequest(order.getCustomerId(), paymentResult.getTransactionId()),
                Void.class
            ).await();

            ctx.complete(new OrderResult(true, paymentResult.getTransactionId()));
        };
    }
}
```

## Registering Workflows with the Runtime

Create a configuration class to register workflows and activities:

```java
import io.dapr.workflows.runtime.WorkflowRuntime;
import io.dapr.workflows.runtime.WorkflowRuntimeBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class WorkflowConfig {

    @Bean
    public WorkflowRuntime workflowRuntime() {
        WorkflowRuntimeBuilder builder = new WorkflowRuntimeBuilder();
        builder.registerWorkflow(OrderFulfillmentWorkflow.class);
        builder.registerActivity(ReserveInventoryActivity.class);
        builder.registerActivity(ChargeCustomerActivity.class);
        builder.registerActivity(ReleaseInventoryActivity.class);
        builder.registerActivity(NotifyCustomerActivity.class);

        WorkflowRuntime runtime = builder.build();
        runtime.start(false); // non-blocking
        return runtime;
    }
}
```

## Starting and Monitoring Workflows

Use the Dapr client to start and query workflow instances:

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<Map<String, String>> placeOrder(@RequestBody OrderRequest order) {
        try (DaprClient client = new DaprClientBuilder().build()) {
            String instanceId = "order-" + order.getOrderId();

            client.startWorkflow(
                "dapr",                             // workflow component
                OrderFulfillmentWorkflow.class.getName(),
                instanceId,
                order,                              // input
                null                                // options
            ).block();

            return ResponseEntity.ok(Map.of("instanceId", instanceId));
        }
    }

    @GetMapping("/{instanceId}/status")
    public ResponseEntity<?> getStatus(@PathVariable String instanceId) {
        try (DaprClient client = new DaprClientBuilder().build()) {
            WorkflowInstanceStatus status = client.getWorkflowInstanceStatus(
                instanceId, "dapr", true).block();

            return ResponseEntity.ok(Map.of(
                "status", status.getRuntimeStatus().toString(),
                "createdAt", status.getCreatedAt().toString()
            ));
        }
    }
}
```

## Parallel Fan-Out with Multiple Activities

```java
// In a workflow - run activities in parallel
Task<InventoryResult> inventoryTask = ctx.callActivity(
    ReserveInventoryActivity.class.getName(), inventoryRequest, InventoryResult.class);

Task<NotificationResult> notificationTask = ctx.callActivity(
    SendNotificationActivity.class.getName(), notifRequest, NotificationResult.class);

// Wait for all to complete
ctx.allOf(List.of(inventoryTask, notificationTask)).await();
```

## Running the Application

```bash
dapr run --app-id order-service \
         --app-port 8080 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- java -jar target/order-service.jar
```

## Summary

Dapr Workflows with the Java SDK provide a powerful way to orchestrate distributed business processes. Define activities as simple Java classes, compose them in workflow orchestrators, and rely on Dapr's runtime for durability, retries, and state management. The Spring Boot integration makes it straightforward to wire up the workflow runtime alongside your existing application components.
