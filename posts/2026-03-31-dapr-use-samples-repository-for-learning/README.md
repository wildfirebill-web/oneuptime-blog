# How to Use Dapr Samples Repository for Learning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Learning, Sample, Quickstart, Tutorial

Description: Learn how to use the official Dapr quickstarts and samples repositories to understand building blocks through runnable, hands-on examples.

---

The fastest way to learn Dapr is by running working examples. The Dapr project maintains two key repositories: `dapr/quickstarts` for focused single-feature demos and `dapr/samples` for community-contributed multi-feature apps.

## Cloning the Quickstarts Repository

```bash
git clone https://github.com/dapr/quickstarts.git
cd quickstarts
```

The quickstarts directory structure maps to Dapr building blocks:

```text
quickstarts/
  tutorials/          - Multi-step learning paths
  service_invocation/ - Direct service-to-service calls
  pub_sub/            - Pub/sub messaging
  state_management/   - State store operations
  bindings/           - Input/output bindings
  secrets_management/ - Secret store access
  configuration/      - Configuration API
  actors/             - Virtual actors
  workflow/           - Workflow orchestration
  observability/      - Tracing and metrics
```

## Running a Quickstart

Each quickstart has self-contained instructions. Start with pub/sub as a representative example:

```bash
cd quickstarts/pub_sub/python/sdk
pip install -r requirements.txt

# Terminal 1 - start the checkout service
dapr run --app-id checkout --app-port 6002 -- python3 checkout.py

# Terminal 2 - start the order processor
dapr run --app-id order-processor --app-port 6001 -- python3 app.py
```

Watch messages flow between services in the terminal output without any message broker configuration in your code.

## Exploring SDK Variants

Most quickstarts provide examples in multiple languages and communication styles:

```bash
ls quickstarts/service_invocation/
# go/      java/      javascript/      python/      dotnet/
# Each has: http/ (raw HTTP) and sdk/ (language SDK) subdirectories
```

Use the `http` variant to understand the underlying API calls, then switch to the `sdk` variant to see idiomatic SDK usage.

## Using the Dapr Samples Repository

The samples repository has community-contributed examples covering advanced patterns:

```bash
git clone https://github.com/dapr/samples.git
ls samples/
```

Notable samples include:

- `distributed-calculator` - Multi-language microservices
- `hello-kubernetes` - Kubernetes deployment walkthrough
- `dapr-traffic-control` - Fine-grained state and pub/sub demo

## Modifying Samples for Experimentation

Fork and modify samples to experiment safely:

```bash
git clone https://github.com/dapr/quickstarts.git
cd quickstarts/state_management/python/sdk

# Swap the state store from Redis to an in-memory store
cat components/statestore.yaml
# Change spec.type from state.redis to state.in-memory
```

```yaml
spec:
  type: state.in-memory
  version: v1
  metadata: []
```

This pattern lets you isolate component behavior from application logic.

## Running Quickstarts in Kubernetes

```bash
cd quickstarts/pub_sub/deploy
kubectl apply -f .
kubectl get pods -w
```

## Summary

The Dapr quickstarts repository provides runnable examples for every building block in multiple languages and SDK styles. Clone the repo, pick the building block you want to learn, run the self-hosted example first, then experiment by swapping components to deepen your understanding of Dapr's portability.
