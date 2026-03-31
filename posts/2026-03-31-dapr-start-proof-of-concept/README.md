# How to Start a Dapr Proof of Concept

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Proof Of Concept, Getting Started, Evaluation, Local Development

Description: Start a Dapr proof of concept in under an hour by setting up a local environment, selecting a pilot service, and demonstrating key building blocks to stakeholders.

---

## Goals of a Dapr PoC

A proof of concept answers three questions:
1. Can Dapr integrate with our existing tech stack?
2. Does the developer experience meet our standards?
3. What is the actual overhead (latency, memory)?

Keep the scope small - one or two building blocks, one or two services.

## Step 1: Install Prerequisites

```bash
# Install Dapr CLI
brew install dapr/tap/dapr-cli

# Initialize Dapr in self-hosted mode (starts Redis and Zipkin via Docker)
dapr init

# Verify
dapr --version
docker ps | grep dapr
```

## Step 2: Choose a Building Block to Demonstrate

Pick one or two building blocks that solve a real problem your team has:

```yaml
poc_scope:
  service_invocation:
    scenario: "Replace hardcoded HTTP client with Dapr SDK"
    effort: low
    impact: high
  state_management:
    scenario: "Store session data without direct Redis dependency"
    effort: low
    impact: medium
  pub_sub:
    scenario: "Decouple order service from inventory service"
    effort: medium
    impact: high
```

## Step 3: Write a Simple Service

Create a minimal Go service that uses Dapr state management:

```go
package main

import (
    "context"
    "log"
    "net/http"
    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    client, err := dapr.NewClient()
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    http.HandleFunc("/save", func(w http.ResponseWriter, r *http.Request) {
        err := client.SaveState(context.Background(),
            "statestore", "poc-key", []byte(`{"status":"ok"}`), nil)
        if err != nil {
            http.Error(w, err.Error(), 500)
            return
        }
        w.Write([]byte("saved"))
    })

    http.ListenAndServe(":8080", nil)
}
```

## Step 4: Run with Dapr

```bash
# Run with Dapr sidecar attached
dapr run \
  --app-id poc-service \
  --app-port 8080 \
  --dapr-http-port 3500 \
  -- go run main.go
```

Test it:

```bash
curl http://localhost:8080/save
# Response: saved

# Verify via Dapr HTTP API directly
curl http://localhost:3500/v1.0/state/statestore/poc-key
# Response: {"status":"ok"}
```

## Step 5: Measure Overhead

Compare request latency with and without Dapr:

```bash
# Benchmark direct Redis vs Dapr state
hey -n 1000 -c 10 http://localhost:8080/save

# Check sidecar resource usage
docker stats dapr_redis dapr_zipkin
```

## Step 6: Present Findings

Document your PoC results:

```markdown
## PoC Results

- Setup time: 45 minutes (first time)
- Integration with existing Redis: Works with component YAML only
- Latency overhead: +2-5ms per state operation
- Memory overhead: 35 MB per service instance
- Developer feedback: Positive - no Redis client needed
- Blockers: None identified
- Recommendation: Proceed to staging pilot
```

## Summary

A Dapr proof of concept can be completed in under a day using the self-hosted mode initialized by `dapr init`. Focus on one or two building blocks that solve real pain points, measure the latency and memory overhead, and document your findings in a structured report. A successful PoC with clear metrics makes it straightforward to get stakeholder approval for a staging pilot.
