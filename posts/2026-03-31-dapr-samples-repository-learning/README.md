# How to Use Dapr Samples Repository for Learning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Samples, Learning, Quickstart, Tutorial

Description: Learn how to use the official Dapr quickstarts and samples repositories to explore all building blocks with runnable examples across multiple programming languages.

---

## The Dapr Quickstarts Repository

The primary learning resource for Dapr is the quickstarts repository at `github.com/dapr/quickstarts`. It contains self-contained examples for every Dapr building block, each available in multiple languages.

```bash
git clone https://github.com/dapr/quickstarts.git
cd quickstarts
ls
# hello-world/  state_management/  pub_sub/  bindings/
# actors/  secrets_management/  configuration/  workflow/
```

## Running Your First Quickstart

The `hello-world` quickstart is the recommended starting point:

```bash
cd quickstarts/hello-world/go/http

# Start the Dapr sidecar and app
dapr run --app-id hello-dapr \
  --app-port 6001 \
  --dapr-http-port 3500 \
  -- go run main.go
```

In a separate terminal, invoke the service:

```bash
dapr invoke --app-id hello-dapr \
  --method order \
  --verb POST \
  --data '{"orderId":"100"}'
```

## Exploring the State Management Quickstart

The state management quickstart demonstrates save, get, and delete operations:

```bash
cd quickstarts/state_management/go/sdk

# Initialize components (Redis is started automatically in self-hosted mode)
dapr run --app-id order-processor \
  --resources-path ../../../components \
  -- go run .
```

The example code shows idiomatic SDK usage:

```go
client, err := dapr.NewClient()
defer client.Close()

// Save state
err = client.SaveState(ctx, "statestore", "order-1",
    []byte(`{"item":"notebook","quantity":5}`), nil)

// Get state
item, err := client.GetState(ctx, "statestore", "order-1", nil)
fmt.Println(string(item.Value))
```

## Multi-Language Support

Each quickstart provides implementations in:
- Go
- Python
- JavaScript (Node.js)
- .NET (C#)
- Java

Switch between languages to compare idioms:

```bash
# Go version
cd quickstarts/pub_sub/go/sdk

# Python version
cd quickstarts/pub_sub/python/sdk

# Both use the same component YAML in ../../../components/
```

## The Dapr Samples Repository

For more complex scenarios, use the samples repository:

```bash
git clone https://github.com/dapr/samples.git
cd samples
ls
# actor-reminder/  distributed-calculator/  eShop/
# hello-kubernetes/  middleware-oauth-google/
```

The `distributed-calculator` sample shows multiple services interacting:

```bash
cd samples/distributed-calculator
docker-compose up
```

## Using Quickstarts for Team Training

Structure a Dapr learning session using quickstarts:

```yaml
training_plan:
  day_1:
    - quickstart: hello-world
    - quickstart: state_management
  day_2:
    - quickstart: pub_sub
    - quickstart: bindings
  day_3:
    - quickstart: actors
    - quickstart: workflow
```

Each quickstart takes 30-60 minutes to complete and understand.

## Summary

The Dapr quickstarts repository provides runnable examples for every Dapr building block in multiple programming languages. Starting with `hello-world` and progressing through state management, pub/sub, actors, and workflows gives developers hands-on experience with the full Dapr API surface. The samples repository complements quickstarts with more realistic multi-service scenarios.
