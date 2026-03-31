# How to Use Dapr Bindings with Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Java, Binding, Integration, Spring Boot, Microservice

Description: Connect Java microservices to external systems like Kafka, cron triggers, and storage using Dapr input and output bindings via the Java SDK and Spring Boot.

---

## Overview

Dapr bindings let Java microservices interact with external infrastructure without importing vendor-specific libraries. Input bindings trigger your service when an event arrives from an external source. Output bindings let your service send data to external targets. The Dapr sidecar handles the connection and protocol details.

## Output Binding: Invoking an External System

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import io.dapr.client.domain.InvokeBindingRequest;

public class BindingExample {
    public static void main(String[] args) throws Exception {
        try (DaprClient client = new DaprClientBuilder().build()) {

            // Send an email via SendGrid binding
            Map<String, Object> emailData = Map.of(
                "emailTo", "user@example.com",
                "subject", "Order Confirmed",
                "body", "Your order ord-1 has been confirmed.");

            client.invokeBinding("sendgrid", "create", emailData).block();
            System.out.println("Email sent");
        }
    }
}
```

## Output Binding to Kafka

Configure the Kafka output binding:

```yaml
# components/kafka-output.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-output
spec:
  type: bindings.kafka
  version: v1
  metadata:
    - name: brokers
      value: "kafka:9092"
    - name: publishTopic
      value: "analytics-events"
```

```java
AnalyticsEvent event = new AnalyticsEvent("page-view", "/products", "user-42");
client.invokeBinding("kafka-output", "create", event).block();
```

## Input Binding: Cron Trigger in Spring Boot

Configure the cron input binding:

```yaml
# components/cron-trigger.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: cron-trigger
spec:
  type: bindings.cron
  version: v1
  metadata:
    - name: schedule
      value: "@every 1m"
```

Handle the cron trigger in a Spring Boot controller:

```java
import org.springframework.web.bind.annotation.*;

@RestController
public class CronController {

    @PostMapping("/cron-trigger")
    public ResponseEntity<Void> handleCron(@RequestBody(required = false) byte[] body) {
        System.out.println("Cron triggered at " + java.time.Instant.now());
        // Execute scheduled work
        runScheduledReport();
        return ResponseEntity.ok().build();
    }
}
```

## Input Binding: AWS S3 Upload Trigger

```yaml
# components/s3-input.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: s3-trigger
spec:
  type: bindings.aws.s3
  version: v1
  metadata:
    - name: bucket
      value: "my-uploads-bucket"
    - name: region
      value: "us-east-1"
```

```java
@PostMapping("/s3-trigger")
public ResponseEntity<Void> handleS3Upload(@RequestBody S3Event event) {
    System.out.printf("New file uploaded: %s (%d bytes)%n",
        event.getKey(), event.getSize());
    processUpload(event.getKey());
    return ResponseEntity.ok().build();
}
```

## Binding with Response

Some output bindings return data. Use the typed `invokeBinding` overload:

```java
byte[] result = client.invokeBinding("http-output", "post",
    payload, byte[].class).block();
System.out.println("Response: " + new String(result));
```

## Summary

Dapr bindings in Java decouple your service from vendor-specific SDKs. Output bindings are a single `invokeBinding` call, while input bindings are handled by a Spring Boot `@PostMapping` endpoint whose path matches the component name. This makes adding new integrations - cron triggers, cloud storage events, message queues - a matter of adding a YAML component and a new controller method.
