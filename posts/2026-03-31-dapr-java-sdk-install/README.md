# How to Install and Configure the Dapr Java SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Java, SDK, Installation, Configuration, Microservice

Description: Install the Dapr Java SDK using Maven or Gradle, configure the client connection, and write your first Java microservice that communicates with a Dapr sidecar.

---

## Overview

The Dapr Java SDK provides blocking and reactive (Project Reactor) clients for all Dapr building blocks. It communicates with the local Dapr sidecar over gRPC and supports Spring Boot auto-configuration through a companion starter package.

## Prerequisites

- Java 17 or later
- Maven 3.8+ or Gradle 8+
- Dapr CLI installed and `dapr init` completed

## Maven Setup

Add the Dapr BOM and SDK to your `pom.xml`:

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.dapr</groupId>
      <artifactId>dapr-sdk-bom</artifactId>
      <version>1.12.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>io.dapr</groupId>
    <artifactId>dapr-sdk</artifactId>
  </dependency>
  <!-- For Spring Boot integration -->
  <dependency>
    <groupId>io.dapr</groupId>
    <artifactId>dapr-sdk-springboot</artifactId>
  </dependency>
</dependencies>
```

## Gradle Setup

```groovy
// build.gradle
dependencies {
    implementation platform('io.dapr:dapr-sdk-bom:1.12.0')
    implementation 'io.dapr:dapr-sdk'
    implementation 'io.dapr:dapr-sdk-springboot'
}
```

## Creating a Blocking Client

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;

public class Main {
    public static void main(String[] args) throws Exception {
        try (DaprClient client = new DaprClientBuilder().build()) {
            // Save state
            client.saveState("statestore", "greeting", "Hello Dapr!").block();

            // Get state
            String value = client.getState("statestore", "greeting", String.class)
                .block()
                .getValue();
            System.out.println("Value: " + value);
        }
    }
}
```

## Creating a Reactive Client

The Dapr Java SDK uses Project Reactor internally. Reactive code avoids blocking threads:

```java
DaprClient client = new DaprClientBuilder().build();

client.saveState("statestore", "key1", "value1")
    .then(client.getState("statestore", "key1", String.class))
    .map(state -> state.getValue())
    .doOnSuccess(v -> System.out.println("Got: " + v))
    .block(); // block only at the entry point
```

## Spring Boot Auto-Configuration

With `dapr-sdk-springboot`, inject `DaprClient` directly into Spring beans:

```java
import io.dapr.client.DaprClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    @Autowired
    private DaprClient daprClient;

    public void saveOrder(Order order) {
        daprClient.saveState("statestore", order.getId(), order).block();
    }
}
```

## Running with the Dapr CLI

```bash
dapr run --app-id java-service --app-port 8080 --components-path ./components \
  -- java -jar target/myapp.jar
```

## Summary

The Dapr Java SDK installs as a standard Maven/Gradle dependency with an optional BOM for version management. The `DaprClientBuilder` auto-discovers the sidecar via environment variables. For Spring Boot applications, the `dapr-sdk-springboot` starter enables `@Autowired DaprClient` with zero configuration, making all Dapr building blocks available anywhere in the application context.
