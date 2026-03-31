# How to Use Dapr Secrets Management with Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Secret, Java, Security, Microservice

Description: Learn how to retrieve and manage secrets in Java applications using Dapr's Secrets Management API with various secret store backends.

---

## Introduction

Dapr's Secrets Management building block provides a unified API for retrieving secrets from providers like Kubernetes secrets, HashiCorp Vault, Azure Key Vault, and AWS Secrets Manager. Your Java code stays the same regardless of which backend you use.

## Adding the Dependency

```xml
<dependency>
  <groupId>io.dapr</groupId>
  <artifactId>dapr-sdk</artifactId>
  <version>1.13.0</version>
</dependency>
```

## Configuring a Secret Store Component

For Kubernetes secrets:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kubernetes-secrets
spec:
  type: secretstores.kubernetes
  version: v1
```

For HashiCorp Vault:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: vault-secrets
spec:
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    - name: vaultAddr
      value: http://vault:8200
    - name: vaultToken
      value: my-vault-token
```

## Retrieving a Single Secret

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import java.util.Map;

DaprClient client = new DaprClientBuilder().build();

Map<String, String> secret = client
    .getSecret("kubernetes-secrets", "db-password", null)
    .block();

String password = secret.get("db-password");
System.out.println("Retrieved secret successfully");
```

## Retrieving a Multi-Value Secret

Some stores support secrets with multiple key-value pairs:

```java
Map<String, String> dbSecrets = client
    .getSecret("vault-secrets", "database/credentials", null)
    .block();

String username = dbSecrets.get("username");
String password = dbSecrets.get("password");
String host = dbSecrets.get("host");
```

## Getting All Secrets from a Store

Retrieve a bulk listing of secrets:

```java
Map<String, Map<String, String>> allSecrets = client
    .getBulkSecret("kubernetes-secrets", null)
    .block();

allSecrets.forEach((name, values) -> {
    System.out.println("Secret: " + name);
    values.forEach((k, v) -> System.out.println("  " + k + "=***"));
});
```

## Using Secrets in Spring Boot

Integrate secrets retrieval into Spring Boot startup:

```java
@Configuration
public class DatabaseConfig {

    @Autowired
    private DaprClient daprClient;

    @Bean
    public DataSource dataSource() {
        Map<String, String> secrets = daprClient
            .getSecret("kubernetes-secrets", "db-credentials", null)
            .block();

        return DataSourceBuilder.create()
            .url("jdbc:postgresql://" + secrets.get("host") + ":5432/mydb")
            .username(secrets.get("username"))
            .password(secrets.get("password"))
            .build();
    }
}
```

## Passing Metadata for Namespace Filtering

Some stores accept metadata to filter or namespace secrets:

```java
Map<String, String> meta = Map.of("namespace", "production");

Map<String, String> secret = client
    .getSecret("vault-secrets", "api-key", meta)
    .block();
```

## Summary

Dapr's Secrets Management API in the Java SDK provides a consistent way to retrieve secrets from any supported backend without changing application code. By configuring the secret store component separately from application logic, you can switch providers or rotate credentials without redeploying your services.
