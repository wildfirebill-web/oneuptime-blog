# How to Use Dapr Configuration with Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Go, Configuration, Feature Flag, Microservice, SDK

Description: Read and subscribe to dynamic configuration values in Go applications using the Dapr Configuration building block backed by Redis or Kubernetes ConfigMaps.

---

## Overview

The Dapr Configuration building block provides a versioned, read-only key-value store for application settings and feature flags. Unlike secrets, configuration values are not sensitive and can change at runtime. The Go SDK supports both point-in-time reads and change subscriptions.

## Configuring the Configuration Store

```yaml
# components/config-store.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: config-store
spec:
  type: configuration.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"
    - name: redisPassword
      value: ""
```

## Reading Configuration Items

```go
package main

import (
    "context"
    "fmt"
    "log"

    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    client, err := dapr.NewClient()
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    ctx := context.Background()

    // Get a single configuration item
    item, err := client.GetConfigurationItem(ctx, "config-store", "max-connections")
    if err != nil {
        log.Fatalf("failed to get config: %v", err)
    }
    fmt.Printf("max-connections = %s (version: %s)\n", item.Value, item.Version)
}
```

## Getting Multiple Items

```go
items, err := client.GetConfigurationItems(ctx, "config-store",
    []string{"max-connections", "feature-flag-new-ui", "log-level"})
if err != nil {
    log.Fatal(err)
}

for key, item := range items {
    fmt.Printf("%s = %s\n", key, item.Value)
}
```

## Subscribing to Configuration Changes

```go
subscriptionID, err := client.SubscribeConfigurationItems(
    ctx,
    "config-store",
    []string{"feature-flag-new-ui", "log-level"},
    func(id string, items map[string]*dapr.ConfigurationItem) {
        for key, item := range items {
            fmt.Printf("Config changed - %s = %s (version: %s)\n",
                key, item.Value, item.Version)
        }
    },
)
if err != nil {
    log.Fatalf("subscription failed: %v", err)
}

// Keep running until signaled
log.Printf("Subscribed with ID: %s", subscriptionID)

// Unsubscribe when done
if err := client.UnsubscribeConfigurationItems(ctx, "config-store", subscriptionID); err != nil {
    log.Printf("unsubscribe error: %v", err)
}
```

## Seeding Configuration in Redis

```bash
redis-cli set max-connections '100'
redis-cli set feature-flag-new-ui 'true'
redis-cli set log-level 'info'
```

## Using Configuration for Feature Flags

```go
type AppConfig struct {
    MaxConnections  int
    EnableNewUI     bool
    LogLevel        string
}

func loadConfig(client dapr.Client, ctx context.Context) (*AppConfig, error) {
    items, err := client.GetConfigurationItems(ctx, "config-store",
        []string{"max-connections", "feature-flag-new-ui", "log-level"})
    if err != nil {
        return nil, err
    }

    return &AppConfig{
        MaxConnections: parseIntOrDefault(items["max-connections"].Value, 50),
        EnableNewUI:    items["feature-flag-new-ui"].Value == "true",
        LogLevel:       items["log-level"].Value,
    }, nil
}
```

## Summary

The Dapr Go configuration API supports both one-time reads and real-time subscriptions. By using `SubscribeConfigurationItems`, Go services receive configuration updates without polling or restarting. This is particularly useful for feature flags and tunable parameters that need to take effect immediately in a running service.
