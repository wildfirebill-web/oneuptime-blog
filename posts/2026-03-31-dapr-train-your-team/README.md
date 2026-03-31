# How to Train Your Team on Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Training, Team, Learning, Developer Experience

Description: Build a structured Dapr training program for your engineering team using official quickstarts, hands-on labs, and internal workshops that fit different learning styles.

---

## Why Structured Training Matters

Ad-hoc Dapr learning leads to gaps in knowledge, inconsistent usage patterns, and support requests that slow down adoption. A structured program ensures every engineer understands the core concepts and your organization's specific Dapr conventions.

## Training Audience Segments

Different roles need different Dapr knowledge:

```yaml
training_tracks:
  application_developers:
    focus: "SDK usage, building block APIs, local development"
    duration: "2 days"
  platform_engineers:
    focus: "Component config, security, observability, upgrades"
    duration: "3 days"
  engineering_managers:
    focus: "Architecture concepts, adoption benefits, trade-offs"
    duration: "2 hours"
```

## Day 1: Core Concepts and Local Setup

Start every training session with a working local environment:

```bash
# Prerequisites verification script
#!/bin/bash
echo "Checking prerequisites..."
dapr --version || { echo "Install Dapr CLI first"; exit 1; }
docker info > /dev/null 2>&1 || { echo "Install Docker first"; exit 1; }
echo "All prerequisites met!"

# Initialize Dapr
dapr init
dapr run --app-id hello -- echo "Dapr is working"
```

Cover concepts in this order:
1. Sidecar pattern - why it exists
2. Building blocks overview
3. Component configuration
4. Service invocation

## Day 1 Lab: State Management Quickstart

```bash
cd quickstarts/state_management/go/sdk

dapr run --app-id order-processor \
  --resources-path ../../../components \
  -- go run .
```

Have developers modify the code to:
- Add a new key
- Implement GetBulkState
- Add error handling for missing keys

## Day 2: Intermediate Topics

Cover more advanced topics on day 2:

```bash
# Lab: Pub/Sub with multiple subscribers
cd quickstarts/pub_sub/go/sdk

# Terminal 1: Start publisher
dapr run --app-id publisher -- go run publisher/main.go

# Terminal 2: Start subscriber 1
dapr run --app-id subscriber-1 -- go run subscriber/main.go
```

Topics:
- Pub/sub routing with CloudEvents
- Resiliency policies
- Secrets management
- Observability with Zipkin

## Creating Internal Workshops

Build a workshop using your organization's actual services:

```markdown
## Workshop: Migrate the Notification Service

### Goal
Replace direct Redis usage with Dapr state management.

### Steps
1. Clone the notification-service repo
2. Apply component YAML from our template repo
3. Replace redis.Client with dapr.NewClient()
4. Run tests and compare output
5. Check traces in Zipkin

### Success Criteria
- All unit tests pass
- State operations visible in Zipkin
- No direct Redis imports remain
```

## Ongoing Learning Resources

Create an internal learning hub:

```yaml
learning_resources:
  documentation:
    - url: "https://docs.dapr.io"
      description: "Official docs - bookmark this"
    - url: "https://github.com/dapr/quickstarts"
      description: "Hands-on examples"
  internal:
    - "Confluence: Dapr at OurCompany"
    - "Slack: #dapr-help channel"
    - "GitHub: dapr-component-templates repo"
  community:
    - "Discord: discord.dapr.io"
    - "YouTube: Dapr Community Calls"
```

## Measuring Training Effectiveness

Track these metrics after training:

```bash
# How many services have been Dapr-enabled
kubectl get pods --all-namespaces -l dapr.io/app-id --no-headers | wc -l

# Time to add new Dapr-enabled service (should decrease over time)
# Developer survey score (1-10) on Dapr confidence
```

## Summary

Effective Dapr team training uses segmented tracks for developers, platform engineers, and managers. A two-day hands-on program using official quickstarts and internal workshops with real services builds practical competence. Measuring time-to-productivity and developer confidence scores after training helps justify continued investment in structured learning programs.
