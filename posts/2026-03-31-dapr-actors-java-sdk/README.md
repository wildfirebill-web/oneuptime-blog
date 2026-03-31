# How to Build Dapr Actors with Java SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Java, Microservice, Distributed System

Description: Learn how to implement Dapr virtual actors in Java using the Java SDK, including defining actor interfaces, managing state, and hosting actors in a Spring Boot service.

---

## What Are Dapr Actors?

Dapr virtual actors provide a stateful, single-threaded programming model for distributed systems. Each actor instance has its own isolated state and processes one request at a time, eliminating concurrency issues without explicit locks. The Java SDK integrates cleanly with Spring Boot to host actors as services.

## Maven Dependencies

```xml
<dependency>
  <groupId>io.dapr</groupId>
  <artifactId>dapr-sdk-actors</artifactId>
  <version>1.12.0</version>
</dependency>
<dependency>
  <groupId>io.dapr</groupId>
  <artifactId>dapr-sdk-springboot</artifactId>
  <version>1.12.0</version>
</dependency>
```

## Defining the Actor Interface

Actor interfaces must extend `Actor` and annotate methods with `@ActorMethod`:

```java
import io.dapr.actors.ActorMethod;
import io.dapr.actors.ActorType;
import reactor.core.publisher.Mono;

@ActorType(name = "BankAccountActor")
public interface BankAccountActor {

    @ActorMethod(name = "deposit")
    Mono<Double> deposit(Double amount);

    @ActorMethod(name = "withdraw")
    Mono<Boolean> withdraw(Double amount);

    @ActorMethod(name = "getBalance")
    Mono<Double> getBalance();
}
```

## Implementing the Actor

```java
import io.dapr.actors.runtime.AbstractActor;
import io.dapr.actors.runtime.ActorRuntimeContext;
import reactor.core.publisher.Mono;

public class BankAccountActorImpl extends AbstractActor implements BankAccountActor {

    private static final String BALANCE_KEY = "balance";

    public BankAccountActorImpl(ActorRuntimeContext runtimeContext, ActorId id) {
        super(runtimeContext, id);
    }

    @Override
    public Mono<Double> deposit(Double amount) {
        return this.getActorStateManager()
            .getOrDefault(BALANCE_KEY, Double.class, 0.0)
            .flatMap(balance -> {
                double newBalance = balance + amount;
                return this.getActorStateManager()
                    .set(BALANCE_KEY, newBalance)
                    .then(this.getActorStateManager().save())
                    .thenReturn(newBalance);
            });
    }

    @Override
    public Mono<Boolean> withdraw(Double amount) {
        return this.getActorStateManager()
            .getOrDefault(BALANCE_KEY, Double.class, 0.0)
            .flatMap(balance -> {
                if (balance < amount) {
                    return Mono.just(false);
                }
                double newBalance = balance - amount;
                return this.getActorStateManager()
                    .set(BALANCE_KEY, newBalance)
                    .then(this.getActorStateManager().save())
                    .thenReturn(true);
            });
    }

    @Override
    public Mono<Double> getBalance() {
        return this.getActorStateManager()
            .getOrDefault(BALANCE_KEY, Double.class, 0.0);
    }
}
```

## Registering Actors in Spring Boot

```java
import io.dapr.actors.runtime.ActorRuntime;
import io.dapr.springboot.DaprApplication;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ActorServiceApplication {

    public static void main(String[] args) {
        // Register the actor implementation
        ActorRuntime.getInstance().getConfig()
            .setActorIdleTimeout(Duration.ofMinutes(60))
            .setActorScanInterval(Duration.ofSeconds(30));

        ActorRuntime.getInstance().registerActor(BankAccountActorImpl.class);

        SpringApplication.run(ActorServiceApplication.class, args);
    }
}
```

Run with Dapr:

```bash
dapr run --app-id bank-service --app-port 8080 -- java -jar target/actor-service.jar
```

## Calling Actors from a Client

```java
import io.dapr.actors.ActorId;
import io.dapr.actors.client.ActorClient;
import io.dapr.actors.client.ActorProxyBuilder;

try (ActorClient actorClient = new ActorClient()) {
    ActorProxyBuilder<BankAccountActor> builder =
        new ActorProxyBuilder<>(BankAccountActor.class, actorClient);

    BankAccountActor account = builder.build(new ActorId("account-001"));

    // Deposit $500
    Double balance = account.deposit(500.0).block();
    System.out.println("Balance after deposit: " + balance);

    // Withdraw $200
    Boolean success = account.withdraw(200.0).block();
    System.out.println("Withdrawal succeeded: " + success);
}
```

## Summary

The Dapr Java SDK makes building virtual actors straightforward with Spring Boot integration. Define a reactive interface with `@ActorMethod`, implement it with `AbstractActor`, and register at startup. The actor proxy pattern gives clients a clean, type-safe API, while Dapr handles state persistence and placement behind the scenes.
